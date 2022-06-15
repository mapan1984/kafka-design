# KafkaApis

在 `handle` 方法中处理 kafka 请求

以 `handleProduceRequest` 方法为例：

``` scala
      // call the replica manager to append messages to the replicas
      replicaManager.appendRecords(
        timeout = produceRequest.timeout.toLong,
        requiredAcks = produceRequest.acks,
        internalTopicsAllowed = internalTopicsAllowed,
        origin = AppendOrigin.Client,
        entriesPerPartition = authorizedRequestInfo,
        responseCallback = sendResponseCallback,
        recordConversionStatsCallback = processingStatsCallback)
```

1. 将 response 通过 `RequestChannel` 的 `sendResponse` 写回到对应的 `Processor` 的 `responseQueue`，`Processor`  的 `processNewResponses` 会从 `responseQueue` 中取回 response
2. replicaManager
