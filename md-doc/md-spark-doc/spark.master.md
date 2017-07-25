[toc]

---
##Spark.Master
画流程图可以从这里开始
http://blog.csdn.net/wangyaninglm/article/details/52887045

首先从启动脚本看起：
sbin/start-master.sh
    
~~~bash
#这里是命令行传输的参数
ORIGINAL_ARGS="$@"
#这里是环境变量设置的地方
. "${SPARK_HOME}/sbin/spark-config.sh"
#这里是环境变量设置的地方
. "${SPARK_HOME}/bin/load-spark-env.sh"
#设置默认端口
if [ "$SPARK_MASTER_PORT" = "" ]; then
  SPARK_MASTER_PORT=7077
fi
#设置默认host
if [ "$SPARK_MASTER_HOST" = "" ]; then
  case `uname` in
      (SunOS)
	  SPARK_MASTER_HOST="`/usr/sbin/check-hostname | awk '{print $NF}'`"
	  ;;
      (*)
	  SPARK_MASTER_HOST="`hostname -f`"
	  ;;
  esac
fi

#这里是Master的全称
CLASS="org.apache.spark.deploy.master.Master"
    
"${SPARK_HOME}/sbin"/spark-daemon.sh start $CLASS 1 \
  --host $SPARK_MASTER_HOST --port $SPARK_MASTER_PORT --webui-port $SPARK_MASTER_WEBUI_PORT \
  $ORIGINAL_ARGS
~~~

所以我们首先分析Master.scala

Master
---
~~~scala
private[deploy] object Master extends Logging {
  val SYSTEM_NAME = "sparkMaster"
  val ENDPOINT_NAME = "Master"

  def main(argStrings: Array[String]) {
    主要是注册信号处理函数，不详述
    Utils.initDaemon(log)
    创建一个配置对象，详见 SparkConf 分析
    val conf = new SparkConf
    这里主要是解析命令行参数
    val args = new MasterArguments(argStrings, conf)
    主要的逻辑在这个函数里面，这里的参数是从命令行上传过来的
    val (rpcEnv, _, _) = startRpcEnvAndEndpoint(args.host, args.port, args.webUiPort, conf)
    rpcEnv.awaitTermination()
  }

  /**
   * Start the Master and return a three tuple of:
   *   (1) The Master RpcEnv
   *   (2) The web UI bound port
   *   (3) The REST server bound port, if any
   */
  def startRpcEnvAndEndpoint(
      host: String,
      port: Int,
      webUiPort: Int,
      conf: SparkConf): (RpcEnv, Int, Option[Int]) = {
    val securityMgr = new SecurityManager(conf)
    创建RPC通信对象
    val rpcEnv = RpcEnv.create(SYSTEM_NAME, host, port, conf, securityMgr)
    val masterEndpoint = rpcEnv.setupEndpoint(ENDPOINT_NAME,
      new Master(rpcEnv, rpcEnv.address, webUiPort, securityMgr, conf))
    val portsResponse = masterEndpoint.askWithRetry[BoundPortsResponse](BoundPortsRequest)
    (rpcEnv, portsResponse.webUIPort, portsResponse.restPort)
  }
}

~~~

See Also http://www.jianshu.com/p/1ff6e833e2e6
~~~mermaid
graph LR;

A[Master.main]-->|Calls|B[Master.startRpcEnvAndEndpoint];
B-->|Calls|C[RpcEnv.create];
C-->|Calls|D[NettyRpcEnvFactory]
~~~

首先我们需要分析RpcEnv是怎么创建RPC对象的。

RpcEnv
---
~~~java
/**
 * A RpcEnv implementation must have a [[RpcEnvFactory]] implementation with an empty constructor
 * so that it can be created via Reflection.
 */
private[spark] object RpcEnv {
  ......
  def create(
      name: String,		"Master"
      bindAddress: String,		"10.199.201.143"
      advertiseAddress: String,	"10.199.201.143"
      port: Int,		7077
      conf: SparkConf,
      securityManager: SecurityManager,
      clientMode: Boolean  	False
      ): RpcEnv = {
    val config = RpcEnvConfig(conf, name, bindAddress, advertiseAddress, port, securityManager,clientMode)
     
    这里是最需要注意的一行，实际上就是实用的Netty
    new NettyRpcEnvFactory().create(config)
  }
}
~~~

TransportConf
---
实际上，这里的配置都是加上了前缀：spark.rpc.,从getConfKey函数可以看出来
这里主要是看RPC可以配置的参数，从最后面可以看出，默认的连接超时是120s，也就是2分钟：
~~~
  public TransportConf(String module, ConfigProvider conf) {
    this.module = module;
    this.conf = conf;
    SPARK_NETWORK_IO_MODE_KEY = getConfKey("io.mode");
    SPARK_NETWORK_IO_PREFERDIRECTBUFS_KEY = getConfKey("io.preferDirectBufs");
    SPARK_NETWORK_IO_CONNECTIONTIMEOUT_KEY = getConfKey("io.connectionTimeout");
    SPARK_NETWORK_IO_BACKLOG_KEY = getConfKey("io.backLog");
    SPARK_NETWORK_IO_NUMCONNECTIONSPERPEER_KEY =  getConfKey("io.numConnectionsPerPeer");
    SPARK_NETWORK_IO_SERVERTHREADS_KEY = getConfKey("io.serverThreads");
    SPARK_NETWORK_IO_CLIENTTHREADS_KEY = getConfKey("io.clientThreads");
    SPARK_NETWORK_IO_RECEIVEBUFFER_KEY = getConfKey("io.receiveBuffer");
    SPARK_NETWORK_IO_SENDBUFFER_KEY = getConfKey("io.sendBuffer");
    SPARK_NETWORK_SASL_TIMEOUT_KEY = getConfKey("sasl.timeout");
    SPARK_NETWORK_IO_MAXRETRIES_KEY = getConfKey("io.maxRetries");
    SPARK_NETWORK_IO_RETRYWAIT_KEY = getConfKey("io.retryWait");
    SPARK_NETWORK_IO_LAZYFD_KEY = getConfKey("io.lazyFD");
  }
  
private String getConfKey(String suffix) {
    return "spark." + module + "." + suffix;
  }

  /** Connect timeout in milliseconds. Default 120 secs. */
  public int connectionTimeoutMs() {
    long defaultNetworkTimeoutS = JavaUtils.timeStringAsSec(
      conf.get("spark.network.timeout", "120s"));
    long defaultTimeoutMs = JavaUtils.timeStringAsSec(
      conf.get(SPARK_NETWORK_IO_CONNECTIONTIMEOUT_KEY, defaultNetworkTimeoutS + "s")) * 1000;
    return (int) defaultTimeoutMs;
  }

~~~


NettyRpcEnvFactory
---
这里的每一行都是比较重要的代码
~~~java
private[rpc] class NettyRpcEnvFactory extends RpcEnvFactory with Logging {

  def create(config: RpcEnvConfig): RpcEnv = {
    val sparkConf = config.conf
    // Use JavaSerializerInstance in multiple threads is safe. However, if we plan to support
    // KryoSerializer in future, we have to use ThreadLocal to store SerializerInstance
    val javaSerializerInstance =
      new JavaSerializer(sparkConf).newInstance().asInstanceOf[JavaSerializerInstance]
    val nettyEnv =
      new NettyRpcEnv(sparkConf, javaSerializerInstance, config.advertiseAddress,
        config.securityManager)
    if (!config.clientMode) {
      val startNettyRpcEnv: Int => (NettyRpcEnv, Int) = { actualPort =>
        nettyEnv.startServer(config.bindAddress, actualPort)
        (nettyEnv, nettyEnv.address.port)
      }
      try {
        Utils.startServiceOnPort(config.port, startNettyRpcEnv, sparkConf, config.name)._1
      } catch {
        case NonFatal(e) =>
          nettyEnv.shutdown()
          throw e
      }
    }
    nettyEnv
  }
}

-->NettyRpcEnv.startServer

def startServer(bindAddress: String, port: Int): Unit = {
    val bootstraps: java.util.List[TransportServerBootstrap] =
      if (securityManager.isAuthenticationEnabled()) {
        java.util.Arrays.asList(new SaslServerBootstrap(transportConf, securityManager))
      } else {
        java.util.Collections.emptyList()
      }
    server = transportContext.createServer(bindAddress, port, bootstraps)
    dispatcher.registerRpcEndpoint(
      RpcEndpointVerifier.NAME, new RpcEndpointVerifier(this, dispatcher))
  }
  
-->TransportContext.createServer
  public TransportServer createServer(
      String host, int port, List<TransportServerBootstrap> bootstraps) {
    return new TransportServer(this, host, port, rpcHandler, bootstraps);
  }
  
-->TransportServer.TransportServer
TransportServer
public TransportServer(
      TransportContext context,
      String hostToBind,
      int portToBind,
      RpcHandler appRpcHandler,
      List<TransportServerBootstrap> bootstraps) {
    this.context = context;
    this.conf = context.getConf();
    this.appRpcHandler = appRpcHandler;
    this.bootstraps = Lists.newArrayList(Preconditions.checkNotNull(bootstraps));

    try {
      init(hostToBind, portToBind);
    } catch (RuntimeException e) {
      JavaUtils.closeQuietly(this);
      throw e;
    }
    
    
-->TransportServer.init
首先看这个函数：支持NIO和EPOLL两种模式
  /** Creates a Netty EventLoopGroup based on the IOMode. */
  public static EventLoopGroup createEventLoop(IOMode mode, int numThreads, String threadPrefix) {
    ThreadFactory threadFactory = createThreadFactory(threadPrefix);

    switch (mode) {
      case NIO:
        return new NioEventLoopGroup(numThreads, threadFactory);
      case EPOLL:
        return new EpollEventLoopGroup(numThreads, threadFactory);
      default:
        throw new IllegalArgumentException("Unknown io mode: " + mode);
    }
  }
  
private void init(String hostToBind, int portToBind) {

    IOMode ioMode = IOMode.valueOf(conf.ioMode());
    EventLoopGroup bossGroup =
      NettyUtils.createEventLoop(ioMode, conf.serverThreads(), conf.getModuleName() + "-server");
    EventLoopGroup workerGroup = bossGroup;

    PooledByteBufAllocator allocator = NettyUtils.createPooledByteBufAllocator(
      conf.preferDirectBufs(), true /* allowCache */, conf.serverThreads());

    bootstrap = new ServerBootstrap()
      .group(bossGroup, workerGroup)
      .channel(NettyUtils.getServerChannelClass(ioMode))
      .option(ChannelOption.ALLOCATOR, allocator)
      .childOption(ChannelOption.ALLOCATOR, allocator);

    ......
    标准的Netty流程：
    bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
      @Override
      protected void initChannel(SocketChannel ch) throws Exception {
        RpcHandler rpcHandler = appRpcHandler;
        for (TransportServerBootstrap bootstrap : bootstraps) {
          rpcHandler = bootstrap.doBootstrap(ch, rpcHandler);
        }
        context.initializePipeline(ch, rpcHandler);
      }
    });

    InetSocketAddress address = hostToBind == null ?
        new InetSocketAddress(portToBind): new InetSocketAddress(hostToBind, portToBind);
    channelFuture = bootstrap.bind(address);
    channelFuture.syncUninterruptibly();

    port = ((InetSocketAddress) channelFuture.channel().localAddress()).getPort();
    logger.debug("Shuffle server started on port: {}", port);
  }
~~~

* TransportServer.init-->TransportContext.initializePipeline->createChannelHandler


~~~java
public TransportChannelHandler initializePipeline(
      SocketChannel channel,
      RpcHandler channelRpcHandler) {
    try {
      TransportChannelHandler channelHandler = createChannelHandler(channel, channelRpcHandler);
      channel.pipeline()
        .addLast("encoder", encoder)
        .addLast(TransportFrameDecoder.HANDLER_NAME, NettyUtils.createFrameDecoder())
        .addLast("decoder", decoder)
        .addLast("idleStateHandler", new IdleStateHandler(0, 0, conf.connectionTimeoutMs() / 1000))
        // NOTE: Chunks are currently guaranteed to be returned in the order of request, but this
        // would require more logic to guarantee if this were not part of the same event loop.
        .addLast("handler", channelHandler);
      return channelHandler;
    } catch (RuntimeException e) {
      logger.error("Error while initializing Netty pipeline", e);
      throw e;
    }
  }
  
  private TransportChannelHandler createChannelHandler(Channel channel, RpcHandler rpcHandler) {
    TransportResponseHandler responseHandler = new TransportResponseHandler(channel);
    TransportClient client = new TransportClient(channel, responseHandler);
    TransportRequestHandler requestHandler = new TransportRequestHandler(channel, client,
      rpcHandler);
    return new TransportChannelHandler(client, responseHandler, requestHandler,
      conf.connectionTimeoutMs(), closeIdleConnections);
  }
  
~~~

TransportChannelHandler
---
public class TransportChannelHandler extends SimpleChannelInboundHandler<Message>
这里就和Netty对接上了

MessageEncoder
---

MessageDecoder
---
看看上面的那些decoder

NettyRpcEnv
---

~~~java
private val transportContext = new TransportContext(transportConf,
    new NettyRpcHandler(dispatcher, this, streamManager))
这里的NettyRpcHandler就是后面传给Netty的 Handler
~~~



SparkTransportConf object
---
~~~java
private val MAX_DEFAULT_NETTY_THREADS = 8
private def defaultNumThreads(numUsableCores: Int): Int = {
    val availableCores =
      if (numUsableCores > 0) numUsableCores else Runtime.getRuntime.availableProcessors()
    math.min(availableCores, MAX_DEFAULT_NETTY_THREADS)
  }
~~~

NettyRpcHandler
---


