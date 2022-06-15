# Sender

``` java
implements Runnable
```

The background thread that handles the sending of produce requests to the Kafka cluster. This thread makes metadata requests to renew its view of the cluster and then send produce requests to the appropriate nodes.

* KafkaClient
* RecordAccumulator
* ProducerMetadata
* `Map<TopicPartition, List<ProducerBatch>> inFlightBatches`

## 方法

### run

``` java
while (running) {
    try {
        runOnce();
    } catch (Exception e) {
        log.error("Uncaught error in kafka producer I/O thread: ", e);
    }
}
```

### runOnce

``` java
/**
 * Run a single iteration of sending
 *
 */
void runOnce() {
    if (transactionManager != null) {
        try {
            transactionManager.maybeResolveSequences();

            // do not continue sending if the transaction manager is in a failed state
            if (transactionManager.hasFatalError()) {
                RuntimeException lastError = transactionManager.lastError();
                if (lastError != null)
                    maybeAbortBatches(lastError);
                client.poll(retryBackoffMs, time.milliseconds());
                return;
            }

            // Check whether we need a new producerId. If so, we will enqueue an InitProducerId
            // request which will be sent below
            transactionManager.bumpIdempotentEpochAndResetIdIfNeeded();

            if (maybeSendAndPollTransactionalRequest()) {
                return;
            }
        } catch (AuthenticationException e) {
            // This is already logged as error, but propagated here to perform any clean ups.
            log.trace("Authentication exception while processing transactional request", e);
            transactionManager.authenticationFailed(e);
        }
    }

    long currentTimeMs = time.milliseconds();
    // 将 record batch 转移到每个节点的生产请求列表中
    long pollTimeout = sendProducerData(currentTimeMs);
    // `NetworkClient.poll` Do actual reads and writes to sockets
    client.poll(pollTimeout, currentTimeMs);
}
```

### sendProducerData

``` java
private long sendProducerData(long now) {
    // 1. 获取 metadata
    Cluster cluster = metadata.fetch();
    // get the list of partitions with data ready to send
    // 2. 从 accumulator 的 batchers 中取出可以发送的数据，
    //     1. 获取对应的 leader 节点列表
    //     2. 找不到 leader 的 topic 列表
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

    // if there are any partitions whose leaders are not known yet, force metadata update
    if (!result.unknownLeaderTopics.isEmpty()) {
        // The set of topics with unknown leader contains topics with leader election pending as well as
        // topics which may have expired. Add the topic again to metadata to ensure it is included
        // and request metadata update, since there are messages to send to the topic.
        for (String topic : result.unknownLeaderTopics)
            this.metadata.add(topic, now);

        log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}",
            result.unknownLeaderTopics);
        this.metadata.requestUpdate();
    }

    // remove any nodes we aren't ready to send to
    Iterator<Node> iter = result.readyNodes.iterator();
    long notReadyTimeout = Long.MAX_VALUE;
    // 4. 遍历 leader 节点列表
    while (iter.hasNext()) {
        Node node = iter.next();
        // 4.1 判断 leader 节点是否已经连接并可以发送数据，如果是返回 true，不是则返回 false 并开始进行连接
        if (!this.client.ready(node, now)) {
            iter.remove();
            notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
        }
    }

    // create produce requests
    // 5. 获取每个 leader 对应的可以发送的 `ProducerBatch` 列表
    Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes, this.maxRequestSize, now);
    // 6. 在 Sender 的 inFlightBatches 里记录 topicPartition 对应的 batch 列表
    addToInflightBatches(batches);
    // 7. 如果需要保证消息的顺序，将 topicPartition 加入到 accumulator 的 muted 记录
    if (guaranteeMessageOrder) {
        // Mute all the partitions drained
        for (List<ProducerBatch> batchList : batches.values()) {
            for (ProducerBatch batch : batchList)
                this.accumulator.mutePartition(batch.topicPartition);
        }
    }

    accumulator.resetNextBatchExpiryTime();
    List<ProducerBatch> expiredInflightBatches = getExpiredInflightBatches(now);
    List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(now);
    expiredBatches.addAll(expiredInflightBatches);

    // Reset the producer id if an expired batch has previously been sent to the broker. Also update the metrics
    // for expired batches. see the documentation of @TransactionState.resetIdempotentProducerId to understand why
    // we need to reset the producer id here.
    if (!expiredBatches.isEmpty())
        log.trace("Expired {} batches in accumulator", expiredBatches.size());
    for (ProducerBatch expiredBatch : expiredBatches) {
        String errorMessage = "Expiring " + expiredBatch.recordCount + " record(s) for " + expiredBatch.topicPartition
            + ":" + (now - expiredBatch.createdMs) + " ms has passed since batch creation";
        failBatch(expiredBatch, -1, NO_TIMESTAMP, new TimeoutException(errorMessage), false);
        if (transactionManager != null && expiredBatch.inRetry()) {
            // This ensures that no new batches are drained until the current in flight batches are fully resolved.
            transactionManager.markSequenceUnresolved(expiredBatch);
        }
    }
    sensors.updateProduceRequestMetrics(batches);

    // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
    // loop and try sending more data. Otherwise, the timeout will be the smaller value between next batch expiry
    // time, and the delay time for checking data availability. Note that the nodes may have data that isn't yet
    // sendable due to lingering, backing off, etc. This specifically does not include nodes with sendable data
    // that aren't ready to send since they would cause busy looping.
    long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
    pollTimeout = Math.min(pollTimeout, this.accumulator.nextExpiryTimeMs() - now);
    pollTimeout = Math.max(pollTimeout, 0);
    if (!result.readyNodes.isEmpty()) {
        log.trace("Nodes with data ready to send: {}", result.readyNodes);
        // if some partitions are already ready to be sent, the select time would be 0;
        // otherwise if some partition already has some data accumulated but not ready yet,
        // the select time will be the time difference between now and its linger expiry time;
        // otherwise the select time will be the time difference between now and the metadata expiry time;
        pollTimeout = 0;
    }
    sendProduceRequests(batches, now);
    return pollTimeout;
}
```

### sendProduceRequests

``` java
/**
 * Transfer the record batches into a list of produce requests on a per-node basis
 */
private void sendProduceRequests(Map<Integer, List<ProducerBatch>> collated, long now) {
    for (Map.Entry<Integer, List<ProducerBatch>> entry : collated.entrySet())
        sendProduceRequest(now, entry.getKey(), acks, requestTimeoutMs, entry.getValue());
}
```

### sendProduceRequest

``` java
/**
 * Create a produce request from the given record batches
 */
private void sendProduceRequest(long now, int destination, short acks, int timeout, List<ProducerBatch> batches) {
    if (batches.isEmpty())
        return;

    final Map<TopicPartition, ProducerBatch> recordsByPartition = new HashMap<>(batches.size());

    // find the minimum magic version used when creating the record sets
    byte minUsedMagic = apiVersions.maxUsableProduceMagic();
    for (ProducerBatch batch : batches) {
        if (batch.magic() < minUsedMagic)
            minUsedMagic = batch.magic();
    }
    ProduceRequestData.TopicProduceDataCollection tpd = new ProduceRequestData.TopicProduceDataCollection();
    for (ProducerBatch batch : batches) {
        TopicPartition tp = batch.topicPartition;
        MemoryRecords records = batch.records();

        // down convert if necessary to the minimum magic used. In general, there can be a delay between the time
        // that the producer starts building the batch and the time that we send the request, and we may have
        // chosen the message format based on out-dated metadata. In the worst case, we optimistically chose to use
        // the new message format, but found that the broker didn't support it, so we need to down-convert on the
        // client before sending. This is intended to handle edge cases around cluster upgrades where brokers may
        // not all support the same message format version. For example, if a partition migrates from a broker
        // which is supporting the new magic version to one which doesn't, then we will need to convert.
        if (!records.hasMatchingMagic(minUsedMagic))
            records = batch.records().downConvert(minUsedMagic, 0, time).records();
        ProduceRequestData.TopicProduceData tpData = tpd.find(tp.topic());
        if (tpData == null) {
            tpData = new ProduceRequestData.TopicProduceData().setName(tp.topic());
            tpd.add(tpData);
        }
        tpData.partitionData().add(new ProduceRequestData.PartitionProduceData()
                .setIndex(tp.partition())
                .setRecords(records));
        recordsByPartition.put(tp, batch);
    }

    String transactionalId = null;
    if (transactionManager != null && transactionManager.isTransactional()) {
        transactionalId = transactionManager.transactionalId();
    }

    ProduceRequest.Builder requestBuilder = ProduceRequest.forMagic(minUsedMagic,
            new ProduceRequestData()
                    .setAcks(acks)
                    .setTimeoutMs(timeout)
                    .setTransactionalId(transactionalId)
                    .setTopicData(tpd));
    RequestCompletionHandler callback = response -> handleProduceResponse(response, recordsByPartition, time.milliseconds());

    String nodeId = Integer.toString(destination);
    ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,
            requestTimeoutMs, callback);
    client.send(clientRequest, now);
    log.trace("Sent produce request to {}: {}", nodeId, requestBuilder);
}
```

### wekeup

``` java
/**
 * Wake up the selector associated with this send thread
 */
public void wakeup() {
    this.client.wakeup();
}
```
