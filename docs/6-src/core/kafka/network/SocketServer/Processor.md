# Processor

## 属性

``` scala
// Acceptor 处理得到的已连接请求会记录到这里
private val newConnections = new ArrayBlockingQueue[SocketChannel](connectionQueueSize)
private val inflightResponses = mutable.Map[String, RequestChannel.Response]()
// RequestChannel 调用 sendResponse() 会把响应记录在这里
private val responseQueue = new LinkedBlockingDeque[RequestChannel.Response]()
```

Acceptor 通过 `processor.accept` 将 socketChannel 加入到 processor 的 `newConnections` 中

Processor 数量由 `num.networker.threads` 决定

processor 的 `run` 处理 socketChannel

``` scala
override def run(): Unit = {
  startupComplete()
  try {
    while (isRunning) {
      try {
        // setup any new connections that have been queued up
        // 这里从 newConnections 中取出 channel，并注册到 selector
        configureNewConnections()
        // register any new responses for writing
        // 从 responseQueue 中取出 response，进行 ？处理
        processNewResponses()
        // select keys, 进行实际的读写事件处理
        poll()
        // 遍历每个已经完成接收的 receive，通过 requestChannel.sendRequest(req)，将请求放到 RequestChannel 的 requestQueue 中，供后续处理
        processCompletedReceives()
        // 遍历每个已经完成发送的 send
        processCompletedSends()
        processDisconnected()
        closeExcessConnections()
      } catch {
        // We catch all the throwables here to prevent the processor thread from exiting. We do this because
        // letting a processor exit might cause a bigger impact on the broker. This behavior might need to be
        // reviewed if we see an exception that needs the entire broker to stop. Usually the exceptions thrown would
        // be either associated with a specific socket channel or a bad request. These exceptions are caught and
        // processed by the individual methods above which close the failing channel and continue processing other
        // channels. So this catch block should only ever see ControlThrowables.
        case e: Throwable => processException("Processor got uncaught exception.", e)
      }
    }
  } finally {
    debug(s"Closing selector - processor $id")
    CoreUtils.swallow(closeAll(), this, Level.ERROR)
    shutdownComplete()
  }
}
```
