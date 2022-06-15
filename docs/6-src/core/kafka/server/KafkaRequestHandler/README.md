# KafkaRequestHandler

从 `RequestChannel` 的 `requestQueue` 中取出请求，并通过 `ApiRequestHandler.handle` 进行处理（这里的实现在 `KafkaApis` 中）


``` scala
def run(): Unit = {
  while (!stopped) {
    // We use a single meter for aggregate idle percentage for the thread pool.
    // Since meter is calculated as total_recorded_value / time_window and
    // time_window is independent of the number of threads, each recorded idle
    // time should be discounted by # threads.
    val startSelectTime = time.nanoseconds

    val req = requestChannel.receiveRequest(300)
    val endTime = time.nanoseconds
    val idleTime = endTime - startSelectTime
    aggregateIdleMeter.mark(idleTime / totalHandlerThreads.get)

    req match {
      case RequestChannel.ShutdownRequest =>
        debug(s"Kafka request handler $id on broker $brokerId received shut down command")
        shutdownComplete.countDown()
        return

      case request: RequestChannel.Request =>
        try {
          request.requestDequeueTimeNanos = endTime
          trace(s"Kafka request handler $id on broker $brokerId handling request $request")
          apis.handle(request)
        } catch {
          case e: FatalExitError =>
            shutdownComplete.countDown()
            Exit.exit(e.statusCode)
          case e: Throwable => error("Exception when handling request", e)
        } finally {
          request.releaseBuffer()
        }

      case null => // continue
    }
  }
  shutdownComplete.countDown()
}
```
