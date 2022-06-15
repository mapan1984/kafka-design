# RequestChannel

`RequestChannel` 负责将请求从网络层转接到业务层，以及将业务层的处理结果交付给网络层进而返回给客户端。

每一个 `SocketServer` 只有一个 `RequestChannel` 对象，在 `SocketServer` 中构造。

`RequestChannel` 构造方法中初始化了 `requestQueue`，用来存放网络层接收到的请求，这些请求即将交付给业务层进行处理（`KafkaRequestHandler` 进行处理）。

同时，初始化了 `processors`，在 `SocketServer` 创建 `Processor`  时会将 `Processor` 同步记录到这里，当 `RequestChannel` 调用 `sendResponse()` 时会从这里取 response 对应的 `Processor`  并将 response 返回给对应的 `Processor`，这些 response 即将交付给网络层返回给客户端。

## 属性

``` scala
private val requestQueue = new ArrayBlockingQueue[BaseRequest](queueSize)
// Processor 记录
private val processors = new ConcurrentHashMap[Int, Processor]()
```

## 方法

### addProcesor

### removeProcessor

### sendRequest

``` scala
/** Send a request to be handled, potentially blocking until there is room in the queue for the request */
def sendRequest(request: RequestChannel.Request): Unit = {
  requestQueue.put(request)
}
```

### sendResponse

``` scala
/** Send a response back to the socket server to be sent over the network */
def sendResponse(response: RequestChannel.Response): Unit = {

  if (isTraceEnabled) {
    val requestHeader = response.request.header
    val message = response match {
      case sendResponse: SendResponse =>
        s"Sending ${requestHeader.apiKey} response to client ${requestHeader.clientId} of ${sendResponse.responseSend.size} bytes."
      case _: NoOpResponse =>
        s"Not sending ${requestHeader.apiKey} response to client ${requestHeader.clientId} as it's not required."
      case _: CloseConnectionResponse =>
        s"Closing connection for client ${requestHeader.clientId} due to error during ${requestHeader.apiKey}."
      case _: StartThrottlingResponse =>
        s"Notifying channel throttling has started for client ${requestHeader.clientId} for ${requestHeader.apiKey}"
      case _: EndThrottlingResponse =>
        s"Notifying channel throttling has ended for client ${requestHeader.clientId} for ${requestHeader.apiKey}"
    }
    trace(message)
  }

  response match {
    // We should only send one of the following per request
    case _: SendResponse | _: NoOpResponse | _: CloseConnectionResponse =>
      val request = response.request
      val timeNanos = time.nanoseconds()
      request.responseCompleteTimeNanos = timeNanos
      if (request.apiLocalCompleteTimeNanos == -1L)
        request.apiLocalCompleteTimeNanos = timeNanos
    // For a given request, these may happen in addition to one in the previous section, skip updating the metrics
    case _: StartThrottlingResponse | _: EndThrottlingResponse => ()
  }

  val processor = processors.get(response.processor)
  // The processor may be null if it was shutdown. In this case, the connections
  // are closed, so the response is dropped.
  if (processor != null) {
    processor.enqueueResponse(response)
  }
}
```

### receiveRequest

``` scala
/** Get the next request or block until specified time has elapsed */
def receiveRequest(timeout: Long): RequestChannel.BaseRequest =
  requestQueue.poll(timeout, TimeUnit.MILLISECONDS)

/** Get the next request or block until there is one */
def receiveRequest(): RequestChannel.BaseRequest =
  requestQueue.take()
```
