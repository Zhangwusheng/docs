#Kafka Server

启动脚本命令如下：

* bin/kafka-server-start.sh  [-daemon] server.properties [--override property=value]*"



这个脚本里面会启动 *kafka.Kafka* 这个类,这个函数很简单，启动kafkaServerStartable

###kafka.Kafka

~~~scala
def main(args: Array[String]): Unit = {
    try {
      val serverProps = getPropsFromArgs(args)
      val kafkaServerStartable = KafkaServerStartable.fromProps(serverProps)

      // attach shutdown handler to catch control-c
      Runtime.getRuntime().addShutdownHook(new Thread() {
        override def run() = {
          kafkaServerStartable.shutdown
        }
      })

      kafkaServerStartable.startup
      kafkaServerStartable.awaitShutdown
    }
    catch {
      case e: Throwable =>
        fatal(e)
        System.exit(1)
    }
    
    System.exit(0)
  }
~~~

###KafkaServerStartable
KafkaServerStartable直接启动了KafkaServer：

~~~scala


class KafkaServerStartable(val serverConfig: KafkaConfig) extends Logging {
  private val server = new KafkaServer(serverConfig)

  def startup() {
    try {
      server.startup()
    }
    catch {
      case e: Throwable =>
        fatal("Fatal error during KafkaServerStartable startup. Prepare to shutdown", e)
        // KafkaServer already calls shutdown() internally, so this is purely for logging & the exit code
        System.exit(1)
    }
  }
  
  def shutdown() {
    try {
      server.shutdown()
    }
    catch {
      case e: Throwable =>
        fatal("Fatal error during KafkaServerStable shutdown. Prepare to halt", e)
        // Calling exit() can lead to deadlock as exit() can be called multiple times. Force exit.
        Runtime.getRuntime.halt(1)
    }
  }

  /**
   * Allow setting broker state from the startable.
   * This is needed when a custom kafka server startable want to emit new states that it introduces.
   */
  def setServerState(newState: Byte) {
    server.brokerState.newState(newState)
  }

  def awaitShutdown() = 
    server.awaitShutdown

}
~~~

###KafkaServer

主要成员变量有：

~~~scala
  var apis: KafkaApis = null
  var socketServer: SocketServer = null
  var requestHandlerPool: KafkaRequestHandlerPool = null
  var logManager: LogManager = null

  var replicaManager: ReplicaManager = null

  var consumerCoordinator: GroupCoordinator = null

  var kafkaController: KafkaController = null

  val kafkaScheduler = new KafkaScheduler(config.backgroundThreads)

  var kafkaHealthcheck: KafkaHealthcheck = null
  val metadataCache: MetadataCache = new MetadataCache(config.brokerId)
~~~


先看startup函数，然后逐个看成员变量是怎么初始化的，然后分析网络协议部分:

~~~scala
/**
   * Start up API for bringing up a single instance of the Kafka server.
   * Instantiates the LogManager, the SocketServer and the request handlers - KafkaRequestHandlers
   */
  def startup() {
    try {
      info("starting")

      if(isShuttingDown.get)
        throw new IllegalStateException("Kafka server is still shutting down, cannot re-start!")

      if(startupComplete.get)
        return

      val canStartup = isStartingUp.compareAndSet(false, true)
      if (canStartup) {
        metrics = new Metrics(metricConfig, reporters, kafkaMetricsTime, true)

        brokerState.newState(Starting)

        /* start scheduler */
        kafkaScheduler.startup()

        /* setup zookeeper */
        zkUtils = initZk()

        /* start log manager */
        logManager = createLogManager(zkUtils.zkClient, brokerState)
        logManager.startup()

        /* generate brokerId */
        config.brokerId =  getBrokerId
        this.logIdent = "[Kafka Server " + config.brokerId + "], "

        socketServer = new SocketServer(config, metrics, kafkaMetricsTime)
        socketServer.startup()

        /* start replica manager */
        replicaManager = new ReplicaManager(config, metrics, time, kafkaMetricsTime, zkUtils, kafkaScheduler, logManager,
          isShuttingDown)
        replicaManager.startup()

        /* start kafka controller */
        kafkaController = new KafkaController(config, zkUtils, brokerState, kafkaMetricsTime, metrics, threadNamePrefix)
        kafkaController.startup()

        /* start kafka coordinator */
        consumerCoordinator = GroupCoordinator.create(config, zkUtils, replicaManager)
        consumerCoordinator.startup()

        /* Get the authorizer and initialize it if one is specified.*/
        authorizer = Option(config.authorizerClassName).filter(_.nonEmpty).map { authorizerClassName =>
          val authZ = CoreUtils.createObject[Authorizer](authorizerClassName)
          authZ.configure(config.originals())
          authZ
        }

        /* start processing requests */
        apis = new KafkaApis(socketServer.requestChannel, replicaManager, consumerCoordinator,
          kafkaController, zkUtils, config.brokerId, config, metadataCache, metrics, authorizer)
        requestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.requestChannel, apis, config.numIoThreads)
        brokerState.newState(RunningAsBroker)

        Mx4jLoader.maybeLoad()

        /* start dynamic config manager */
        dynamicConfigHandlers = Map[String, ConfigHandler](ConfigType.Topic -> new TopicConfigHandler(logManager),
                                                           ConfigType.Client -> new ClientIdConfigHandler(apis.quotaManagers))

        // Apply all existing client configs to the ClientIdConfigHandler to bootstrap the overrides
        // TODO: Move this logic to DynamicConfigManager
        AdminUtils.fetchAllEntityConfigs(zkUtils, ConfigType.Client).foreach {
          case (clientId, properties) => dynamicConfigHandlers(ConfigType.Client).processConfigChanges(clientId, properties)
        }

        // Create the config manager. start listening to notifications
        dynamicConfigManager = new DynamicConfigManager(zkUtils, dynamicConfigHandlers)
        dynamicConfigManager.startup()

        /* tell everyone we are alive */
        val listeners = config.advertisedListeners.map {case(protocol, endpoint) =>
          if (endpoint.port == 0)
            (protocol, EndPoint(endpoint.host, socketServer.boundPort(protocol), endpoint.protocolType))
          else
            (protocol, endpoint)
        }
        kafkaHealthcheck = new KafkaHealthcheck(config.brokerId, listeners, zkUtils)
        kafkaHealthcheck.startup()

        /* register broker metrics */
        registerStats()

        shutdownLatch = new CountDownLatch(1)
        startupComplete.set(true)
        isStartingUp.set(false)
        AppInfoParser.registerAppInfo(jmxPrefix, config.brokerId.toString)
        info("started")
      }
    }
    catch {
      case e: Throwable =>
        fatal("Fatal error during KafkaServer startup. Prepare to shutdown", e)
        isStartingUp.set(false)
        shutdown()
        throw e
    }
  }
~~~


##SocketServer

先看 这个类的主要注释：

~~~scala
/**
 * An NIO socket server. The threading model is
 *   1 Acceptor thread that handles new connections
 *   Acceptor has N Processor threads that each have their own selector and read requests from sockets
 *   M Handler threads that handle requests and produce responses back to the processor threads for writing.
 */
~~~



* 变量numProcessorThreads

~~~scala
val NumNetworkThreadsProp = "num.network.threads"
val numNetworkThreads = getInt(KafkaConfig.NumNetworkThreadsProp)
private val numProcessorThreads = config.numNetworkThreads
~~~

可以看出，numProcessorThreads就是  "num.network.threads" 这个配置项。

* 变量maxQueuedRequests

~~~scala
private val maxQueuedRequests = config.queuedMaxRequests
val queuedMaxRequests = getInt(KafkaConfig.QueuedMaxRequestsProp)
val QueuedMaxRequestsProp = "queued.max.requests"
~~~

* 变量listeners

~~~scala
  val PortProp = "port"
  val HostNameProp = "host.name"
  val ListenersProp = "listeners"
  
  -------
  
  val hostName = getString(KafkaConfig.HostNameProp)
  val port = getInt(KafkaConfig.PortProp)
  
  -------
    // If the user did not define listeners but did define host or port, let's use them in backward compatible way
  // If none of those are defined, we default to PLAINTEXT://:9092
  private def getListeners(): immutable.Map[SecurityProtocol, EndPoint] = {
    if (getString(KafkaConfig.ListenersProp) != null) {
      validateUniquePortAndProtocol(getString(KafkaConfig.ListenersProp))
      CoreUtils.listenerListToEndPoints(getString(KafkaConfig.ListenersProp))
    } else {
      CoreUtils.listenerListToEndPoints("PLAINTEXT://" + hostName + ":" + port)
    }
  }
  
  val listeners = getListeners
~~~

~~~scala
  def listenerListToEndPoints(listeners: String): immutable.Map[SecurityProtocol, EndPoint] = {
    val listenerList = parseCsvList(listeners)
    listenerList.map(listener => EndPoint.createEndPoint(listener)).map(ep => ep.protocolType -> ep).toMap
  }
  
  def createEndPoint(connectionString: String): EndPoint = {
    val uriParseExp = """^(.*)://\[?([0-9a-zA-Z\-.:]*)\]?:(-?[0-9]+)""".r
    connectionString match {
      case uriParseExp(protocol, "", port) => new EndPoint(null, port.toInt, SecurityProtocol.valueOf(protocol))
      case uriParseExp(protocol, host, port) => new EndPoint(host, port.toInt, SecurityProtocol.valueOf(protocol))
      case _ => throw new KafkaException("Unable to parse " + connectionString + " to a broker endpoint")
    }
  }
  
  
  
  public enum SecurityProtocol {
    /** Un-authenticated, non-encrypted channel */
    PLAINTEXT(0, "PLAINTEXT", false),
    
~~~

一般listeners配置为空，所以会根据 PLAINTEXT://"host.name"."port"来创建一个EndPoint。
顺便看看EndPoint是怎么序列化的：

~~~scala
/**
 * Part of the broker definition - matching host/port pair to a protocol
 */
case class EndPoint(host: String, port: Int, protocolType: SecurityProtocol) {

  def connectionString(): String = {
    val hostport =
      if (host == null)
        ":"+port
      else
        Utils.formatAddress(host, port)
    protocolType + "://" + hostport
  }

  def writeTo(buffer: ByteBuffer): Unit = {
    buffer.putInt(port)
    writeShortString(buffer, host)
    buffer.putShort(protocolType.id)
  }

  def sizeInBytes: Int =
    4 + /* port */
    shortStringLength(host) +
    2 /* protocol id */
}
~~~


* 变量totalProcessorThreads

~~~scala
private val totalProcessorThreads = numProcessorThreads * endpoints.size
~~~


##RequestChannel
很明显，这个是处理网络请求的

~~~scala
val requestChannel = new RequestChannel(totalProcessorThreads, maxQueuedRequests)
~~~


类定义：定义了请求队列和响应队列，请求队列大小为queueSize，responseQueues为numProcessors个，一般对应于CPU数目。一个Processor一个响应队列。这些Processor就是按照0-numProcessors进行编号的。

~~~scala
class RequestChannel(val numProcessors: Int, val queueSize: Int) extends KafkaMetricsGroup {
  private var responseListeners: List[(Int) => Unit] = Nil
  private val requestQueue = new ArrayBlockingQueue[RequestChannel.Request](queueSize)
  private val responseQueues = new Array[BlockingQueue[RequestChannel.Response]](numProcessors)
  for(i <- 0 until numProcessors)
    responseQueues(i) = new LinkedBlockingQueue[RequestChannel.Response]()

~~~

其他的函数其实也很简单，就是提供了Request和Response的put和get操作

~~~scala
  /** Send a request to be handled, potentially blocking until there is room in the queue for the request */
  def sendRequest(request: RequestChannel.Request) {
    requestQueue.put(request)
  }
  
  /** Send a response back to the socket server to be sent over the network */ 
  def sendResponse(response: RequestChannel.Response) {
    responseQueues(response.processor).put(response)
    for(onResponse <- responseListeners)
      onResponse(response.processor)
  }

  /** No operation to take for the request, need to read more over the network */
  def noOperation(processor: Int, request: RequestChannel.Request) {
    responseQueues(processor).put(new RequestChannel.Response(processor, request, null, RequestChannel.NoOpAction))
    for(onResponse <- responseListeners)
      onResponse(processor)
  }

  /** Close the connection for the request */
  def closeConnection(processor: Int, request: RequestChannel.Request) {
    responseQueues(processor).put(new RequestChannel.Response(processor, request, null, RequestChannel.CloseConnectionAction))
    for(onResponse <- responseListeners)
      onResponse(processor)
  }

  /** Get the next request or block until specified time has elapsed */
  def receiveRequest(timeout: Long): RequestChannel.Request =
    requestQueue.poll(timeout, TimeUnit.MILLISECONDS)

  /** Get the next request or block until there is one */
  def receiveRequest(): RequestChannel.Request =
    requestQueue.take()

  /** Get a response for the given processor if there is one */
  def receiveResponse(processor: Int): RequestChannel.Response = {
    val response = responseQueues(processor).poll()
    if (response != null)
      response.request.responseDequeueTimeMs = SystemTime.milliseconds
    response
  }

  def addResponseListener(onResponse: Int => Unit) { 
    responseListeners ::= onResponse
  }

  def shutdown() {
    requestQueue.clear
  }
~~~

看看Session/Request/Response是怎么定义的：
首先获取requestId，是一个short，然后把这个request序列化出来。

~~~scala
 case class Session(principal: KafkaPrincipal, clientAddress: InetAddress)

  case class Request(processor: Int, connectionId: String, session: Session, private var buffer: ByteBuffer, startTimeMs: Long, securityProtocol: SecurityProtocol) {
    // These need to be volatile because the readers are in the network thread and the writers are in the request
    // handler threads or the purgatory threads
    @volatile var requestDequeueTimeMs = -1L
    @volatile var apiLocalCompleteTimeMs = -1L
    @volatile var responseCompleteTimeMs = -1L
    @volatile var responseDequeueTimeMs = -1L
    @volatile var apiRemoteCompleteTimeMs = -1L

    val requestId = buffer.getShort()
    
    
    // for server-side request / response format
    // TODO: this will be removed once we migrated to client-side format
    val requestObj =
      if ( RequestKeys.keyToNameAndDeserializerMap.contains(requestId))
        RequestKeys.deserializerForKey(requestId)(buffer)
      else
        null
        
       
    // if we failed to find a server-side mapping, then try using the
    // client-side request / response format
    val header: RequestHeader =
      if (requestObj == null) {
        buffer.rewind
        RequestHeader.parse(buffer)
      } else
        null
        
        
    val body: AbstractRequest =
      if (requestObj == null)
        AbstractRequest.getRequest(header.apiKey, header.apiVersion, buffer)
      else
        null


    private def requestDesc(details: Boolean): String = {
      if (requestObj != null)
        requestObj.describe(details)
      else
        header.toString + " -- " + body.toString
    }
}
  
  case class Response(processor: Int, request: Request, responseSend: Send, responseAction: ResponseAction) {
    request.responseCompleteTimeMs = SystemTime.milliseconds

    def this(processor: Int, request: Request, responseSend: Send) =
      this(processor, request, responseSend, if (responseSend == null) NoOpAction else SendAction)

    def this(request: Request, send: Send) =
      this(request.processor, request, send)
  }
~~~


主要看看一个请求是怎么序列化的：

~~~scala
object RequestKeys {
  val ProduceKey: Short = 0
  val FetchKey: Short = 1
  val OffsetsKey: Short = 2
  val MetadataKey: Short = 3
  val LeaderAndIsrKey: Short = 4
  val StopReplicaKey: Short = 5
  val UpdateMetadataKey: Short = 6
  val ControlledShutdownKey: Short = 7
  val OffsetCommitKey: Short = 8
  val OffsetFetchKey: Short = 9
  val GroupCoordinatorKey: Short = 10
  val JoinGroupKey: Short = 11
  val HeartbeatKey: Short = 12
  val LeaveGroupKey: Short = 13
  val SyncGroupKey: Short = 14
  val DescribeGroupsKey: Short = 15
  val ListGroupsKey: Short = 16

  // NOTE: this map only includes the server-side request/response handlers. Newer
  // request types should only use the client-side versions which are parsed with
  // o.a.k.common.requests.AbstractRequest.getRequest()
  val keyToNameAndDeserializerMap: Map[Short, (String, (ByteBuffer) => RequestOrResponse)]=
    Map(ProduceKey -> ("Produce", ProducerRequest.readFrom),
        FetchKey -> ("Fetch", FetchRequest.readFrom),
        OffsetsKey -> ("Offsets", OffsetRequest.readFrom),
        MetadataKey -> ("Metadata", TopicMetadataRequest.readFrom),
        LeaderAndIsrKey -> ("LeaderAndIsr", LeaderAndIsrRequest.readFrom),
        StopReplicaKey -> ("StopReplica", StopReplicaRequest.readFrom),
        UpdateMetadataKey -> ("UpdateMetadata", UpdateMetadataRequest.readFrom),
        ControlledShutdownKey -> ("ControlledShutdown", ControlledShutdownRequest.readFrom),
        OffsetCommitKey -> ("OffsetCommit", OffsetCommitRequest.readFrom),
        OffsetFetchKey -> ("OffsetFetch", OffsetFetchRequest.readFrom)
    )
~~~



###Processor

~~~scala
private val processors = new Array[Processor](totalProcessorThreads)
~~~


~~~scala
/**
 * Thread that processes all requests from a single connection. There are N of these running in parallel
 * each of which has its own selector
 */
private[kafka] class Processor(val id: Int,
                               time: Time,
                               maxRequestSize: Int,
                               requestChannel: RequestChannel,
                               connectionQuotas: ConnectionQuotas,
                               connectionsMaxIdleMs: Long,
                               protocol: SecurityProtocol,
                               channelConfigs: java.util.Map[String, _],
                               metrics: Metrics) extends AbstractServerThread(connectionQuotas) with KafkaMetricsGroup {
~~~

首先，构造函数里面有RequestChannel，

~~~scala
//保存了新的网络连接
  private val newConnections = new ConcurrentLinkedQueue[SocketChannel]()
//未处理完的响应？
  private val inflightResponses = mutable.Map[String, RequestChannel.Response]()
  
 //这里理解为PlaintextChannelBuilder
  private val channelBuilder = ChannelBuilders.create(protocol, Mode.SERVER, LoginType.SERVER, channelConfigs)
  
  
  private val metricTags = new util.HashMap[String, String]()
  metricTags.put("networkProcessor", id.toString)
  
  //这里定义了网络的Selector，就是Reactor机制,KafkaSelector的缩写，是kafka自己实现的Selector机制，基于nio，这个selector可以好好读一读。 
    private val selector = new KSelector(
    maxRequestSize,
    connectionsMaxIdleMs,
    metrics,
    time,
    "socket-server",
    metricTags,
    false,
    channelBuilder)


override def run() {
	//先分析完Selector再分析这个函数。
}
~~~

##Selector
具体见selector得分析章节。

##Acceptor

具体见Acceptor得分析章节

下面继续分析网络请求是如何被kafka处理的：


系统接收到网络请求后NetwodkRecieve会调用到SocketServer.java:427Processer.run里面的一行：  requestChannel.sendRequest(req)这时候就把Request放到队列里面了。剩下就看怎么处理这些Request了。再看        kafka/server/KafkaServer.scalarequestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.requestChannel, apis, config.numIoThreads)KafkaRequestHandler。run只有一行，会调用apis.handle(req)很显然，我们就正式的回归到kafka的处理上来了




