# 消息写入

`KafkaApis.handleProduceRequest()` 中最终写入消息是通过调用 `replicaManager.appendRecords()` 完成的

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

`
- ReplicaManager.appendRecords()
    - ReplicaManager.appendToLocalLog()
        - Partition.appendRecordsToLeader()
            - Log.appendAsLeader()
`

`ReplicaManager.appendRecords()`


