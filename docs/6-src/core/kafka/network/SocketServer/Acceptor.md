# Acceptor

## 属性

``` scala
private val nioSelector = NSelector.open()
val serverChannel = openServerSocket(endPoint.host, endPoint.port)
// 关联的 Processor 记录
private val processors = new ArrayBuffer[Processor]()
```

## 方法

### run

在 run 中不断尝试处理连接：

``` scala
/**
 * Accept loop that checks for new connection attempts
 */
def run(): Unit = {
  serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
  startupComplete()
  try {
    while (isRunning) {
      try {
        acceptNewConnections()
        closeThrottledConnections()
      }
      catch {
        // We catch all the throwables to prevent the acceptor thread from exiting on exceptions due
        // to a select operation on a specific channel or a bad request. We don't want
        // the broker to stop responding to requests from other clients in these scenarios.
        case e: ControlThrowable => throw e
        case e: Throwable => error("Error occurred", e)
      }
    }
  } finally {
    debug("Closing server socket, selector, and any throttled sockets.")
    CoreUtils.swallow(serverChannel.close(), this, Level.ERROR)
    CoreUtils.swallow(nioSelector.close(), this, Level.ERROR)
    throttledSockets.foreach(throttledSocket => closeSocket(throttledSocket.socket))
    throttledSockets.clear()
    shutdownComplete()
  }
}
```

### acceptNewConnections

在 `acceptNewConnections` 中通过 nio select 筛选可连接的 channel，accept 之，对每个 accepted channel， 尝试在 `processors` 中选择一个 `processor` 并分配之

``` scala
/**
 * Listen for new connections and assign accepted connections to processors using round-robin.
 */
private def acceptNewConnections(): Unit = {
  val ready = nioSelector.select(500)
  if (ready > 0) {
    val keys = nioSelector.selectedKeys()
    val iter = keys.iterator()
    while (iter.hasNext && isRunning) {
      try {
        val key = iter.next
        iter.remove()

        if (key.isAcceptable) {
          accept(key).foreach { socketChannel =>
            // Assign the channel to the next processor (using round-robin) to which the
            // channel can be added without blocking. If newConnections queue is full on
            // all processors, block until the last one is able to accept a connection.
            var retriesLeft = synchronized(processors.length)
            var processor: Processor = null
            do {
              retriesLeft -= 1
              processor = synchronized {
                // adjust the index (if necessary) and retrieve the processor atomically for
                // correct behaviour in case the number of processors is reduced dynamically
                currentProcessorIndex = currentProcessorIndex % processors.length
                processors(currentProcessorIndex)
              }
              currentProcessorIndex += 1
            } while (!assignNewConnection(socketChannel, processor, retriesLeft == 0))
          }
        } else
          throw new IllegalStateException("Unrecognized key state for acceptor thread.")
      } catch {
        case e: Throwable => error("Error while accepting connection", e)
      }
    }
  }
}
```

### assignNewConnection

`assignNewConnection` 通过调用 `processor.accept` 将 socketChannel 加入到 processor 自身维护的 `newConnections` 队列中

``` scala
private def assignNewConnection(socketChannel: SocketChannel, processor: Processor, mayBlock: Boolean): Boolean = {
  if (processor.accept(socketChannel, mayBlock, blockedPercentMeter)) {
    debug(s"Accepted connection from ${socketChannel.socket.getRemoteSocketAddress} on" +
      s" ${socketChannel.socket.getLocalSocketAddress} and assigned it to processor ${processor.id}," +
      s" sendBufferSize [actual|requested]: [${socketChannel.socket.getSendBufferSize}|$sendBufferSize]" +
      s" recvBufferSize [actual|requested]: [${socketChannel.socket.getReceiveBufferSize}|$recvBufferSize]")
    true
  } else
    false
}
```

