# NetworkClient

``` java
implements KafkaClient
```

## 方法

### poll

``` java
/**
 * Do actual reads and writes to sockets.
 *
 * @param timeout The maximum amount of time to wait (in ms) for responses if there are none immediately,
 *                must be non-negative. The actual timeout will be the minimum of timeout, request timeout and
 *                metadata timeout
 * @param now The current time in milliseconds
 * @return The list of responses received
 */
@Override
public List<ClientResponse> poll(long timeout, long now) {
    ensureActive();

    if (!abortedSends.isEmpty()) {
        // If there are aborted sends because of unsupported version exceptions or disconnects,
        // handle them immediately without waiting for Selector#poll.
        List<ClientResponse> responses = new ArrayList<>();
        handleAbortedSends(responses);
        completeResponses(responses);
        return responses;
    }

    long metadataTimeout = metadataUpdater.maybeUpdate(now);
    try {
        this.selector.poll(Utils.min(timeout, metadataTimeout, defaultRequestTimeoutMs));
    } catch (IOException e) {
        log.error("Unexpected error during I/O", e);
    }

    // process completed actions
    long updatedNow = this.time.milliseconds();
    List<ClientResponse> responses = new ArrayList<>();
    handleCompletedSends(responses, updatedNow);
    handleCompletedReceives(responses, updatedNow);
    handleDisconnections(responses, updatedNow);
    handleConnections();
    handleInitiateApiVersionRequests(updatedNow);
    handleTimedOutConnections(responses, updatedNow);
    handleTimedOutRequests(responses, updatedNow);
    completeResponses(responses);

    return responses;
}
```

1. 如果需要更新 Metadata，那么就发送 Metadata 请求
2. 调用 `Selector` 进行相应的 IO 操作
3. 处理 server 端的 response 及一些其他操作

### send

``` java
/**
 * Queue up the given request for sending. Requests can only be sent out to ready nodes.
 * @param request The request
 * @param now The current timestamp
 */
@Override
public void send(ClientRequest request, long now) {
    doSend(request, false, now);
}
```

### doSend

``` java
```

## 内部类

### DefaultMetadataUpdater

#### maybeUpdate

如果 Metadata 需要更新，选择连接数最小的 node 发送 Metadata 请求
