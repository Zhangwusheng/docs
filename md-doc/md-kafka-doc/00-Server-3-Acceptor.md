##Acceptor

无他，就是采用轮训的方式，针对每一个网络连接调用Processor.accept而已。

~~~scala

/**
 * Thread that accepts and configures new connections. There is one of these per endpoint.
 */
private[kafka] class Acceptor(val endPoint: EndPoint,
                              val sendBufferSize: Int,
                              val recvBufferSize: Int,
                              brokerId: Int,
                              processors: Array[Processor],
                              connectionQuotas: ConnectionQuotas) extends AbstractServerThread(connectionQuotas) with KafkaMetricsGroup {

  private val nioSelector = NSelector.open()
  val serverChannel = openServerSocket(endPoint.host, endPoint.port)

  this.synchronized {
    processors.foreach { processor =>
      Utils.newThread("kafka-network-thread-%d-%s-%d".format(brokerId, endPoint.protocolType.toString, processor.id), processor, false).start()
    }
  }

  /**
   * Accept loop that checks for new connection attempts
   */
  def run() {
    serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
    startupComplete()
    try {
      var currentProcessor = 0
      while (isRunning) {
        try {
          val ready = nioSelector.select(500)
          if (ready > 0) {
            val keys = nioSelector.selectedKeys()
            val iter = keys.iterator()
            while (iter.hasNext && isRunning) {
              try {
                val key = iter.next
                iter.remove()
                if (key.isAcceptable)
                  accept(key, processors(currentProcessor))
                else
                  throw new IllegalStateException("Unrecognized key state for acceptor thread.")

                // round robin to the next processor thread
                currentProcessor = (currentProcessor + 1) % processors.length
              } catch {
                case e: Throwable => error("Error while accepting connection", e)
              }
            }
          }
        }
        catch {
          // We catch all the throwables to prevent the acceptor thread from exiting on exceptions due
          // to a select operation on a specific channel or a bad request. We don't want the
          // the broker to stop responding to requests from other clients in these scenarios.
          case e: ControlThrowable => throw e
          case e: Throwable => error("Error occurred", e)
        }
      }
    } finally {
      debug("Closing server socket and selector.")
      swallowError(serverChannel.close())
      swallowError(nioSelector.close())
      shutdownComplete()
    }
  }

  /*
   * Create a server socket to listen for connections on.
   */
  private def openServerSocket(host: String, port: Int): ServerSocketChannel = {
    val socketAddress =
      if(host == null || host.trim.isEmpty)
        new InetSocketAddress(port)
      else
        new InetSocketAddress(host, port)
    val serverChannel = ServerSocketChannel.open()
    serverChannel.configureBlocking(false)
    serverChannel.socket().setReceiveBufferSize(recvBufferSize)
    try {
      serverChannel.socket.bind(socketAddress)
      info("Awaiting socket connections on %s:%d.".format(socketAddress.getHostName, serverChannel.socket.getLocalPort))
    } catch {
      case e: SocketException =>
        throw new KafkaException("Socket server failed to bind to %s:%d: %s.".format(socketAddress.getHostName, port, e.getMessage), e)
    }
    serverChannel
  }

  /*
   * Accept a new connection
   */
  def accept(key: SelectionKey, processor: Processor) {
    val serverSocketChannel = key.channel().asInstanceOf[ServerSocketChannel]
    val socketChannel = serverSocketChannel.accept()
    try {
      connectionQuotas.inc(socketChannel.socket().getInetAddress)
      socketChannel.configureBlocking(false)
      socketChannel.socket().setTcpNoDelay(true)
      socketChannel.socket().setKeepAlive(true)
      socketChannel.socket().setSendBufferSize(sendBufferSize)

      debug("Accepted connection from %s on %s. sendBufferSize [actual|requested]: [%d|%d] recvBufferSize [actual|requested]: [%d|%d]"
            .format(socketChannel.socket.getInetAddress, socketChannel.socket.getLocalSocketAddress,
                  socketChannel.socket.getSendBufferSize, sendBufferSize,
                  socketChannel.socket.getReceiveBufferSize, recvBufferSize))

      processor.accept(socketChannel)
    } catch {
      case e: TooManyConnectionsException =>
        info("Rejected connection from %s, address already has the configured maximum of %d connections.".format(e.ip, e.count))
        close(socketChannel)
    }
  }

  /**
   * Wakeup the thread for selection.
   */
  @Override
  def wakeup = nioSelector.wakeup()

}
~~~


##Processor.accept
这里也仅仅是把他加入到队列里面去而已。wakeup的作用？

~~~scala
  /**
   * Queue up a new connection for reading
   */
  def accept(socketChannel: SocketChannel) {
    newConnections.add(socketChannel)
    wakeup()
  }
~~~


~~~scala
Processor.run:

override def run() {
    startupComplete()
    while(isRunning) {
      try {
        // setup any new connections that have been queued up
        configureNewConnections()
        ...
 }
 
   private def configureNewConnections() {
    while(!newConnections.isEmpty) {
      val channel = newConnections.poll()
      try {
        val localHost = channel.socket().getLocalAddress.getHostAddress
        val localPort = channel.socket().getLocalPort
        val remoteHost = channel.socket().getInetAddress.getHostAddress
        val remotePort = channel.socket().getPort
        val connectionId = ConnectionId(localHost, localPort, remoteHost, remotePort).toString
        
        //这里加入对这个网络的监听读
        selector.register(connectionId, channel)
      } catch {
        ...
      }
    }
  }
  
  
    /**
     * Register the nioSelector with an existing channel
     * Use this on server-side, when a connection is accepted by a different thread but processed by the Selector
     * Note that we are not checking if the connection id is valid - since the connection already exists
     */
    public void register(String id, SocketChannel socketChannel) throws ClosedChannelException {
        SelectionKey key = socketChannel.register(nioSelector, SelectionKey.OP_READ);
        KafkaChannel channel = channelBuilder.buildChannel(id, key, maxReceiveSize);
        
        //这里加入SelectionKey.OP_READ的监听，并且把KafkaChannel作为数据attach上去。
        key.attach(channel);
        this.channels.put(id, channel);
    }

~~~


至此完成了对网络连接的数据监听！
