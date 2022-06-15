# RecordAccumulator

This class acts as a queue that accumulates records into `MemoryRecords` instances to be send to the server.

## 属性

`batches`

``` java
private final ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches
```

```
{
    // TopicPartition: Deque<ProducerBatch>
}
```

## 方法

### append

将消息加入 `batches`，`batches` 的结构如上

``` java
public RecordAppendResult append(TopicPartition tp,
                                 long timestamp,
                                 byte[] key,
                                 byte[] value,
                                 Header[] headers,
                                 Callback callback,
                                 long maxTimeToBlock,
                                 boolean abortOnNewBatch,
                                 long nowMs) throws InterruptedException {
    // We keep track of the number of appending thread to make sure we do not miss batches in
    // abortIncompleteBatches().
    appendsInProgress.incrementAndGet();
    ByteBuffer buffer = null;
    if (headers == null) headers = Record.EMPTY_HEADERS;
    try {
        // check if we have an in-progress batch
        Deque<ProducerBatch> dq = getOrCreateDeque(tp);
        synchronized (dq) {
            if (closed)
                throw new KafkaException("Producer closed while send in progress");
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq, nowMs);  // 追加数据
            if (appendResult != null)  // topic-partition 已经追加到存在的 batch，直接返回
                return appendResult;
        }

        // 否则尝试创建新的 batch


        // we don't have an in-progress record batch try to allocate a new batch
        if (abortOnNewBatch) {
            // Return a result that will cause another call to append.
            return new RecordAppendResult(null, false, false, true);
        }

        byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
        int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
        log.trace("Allocating a new {} byte message buffer for topic {} partition {} with remaining timeout {}ms", size, tp.topic(), tp.partition(), maxTimeToBlock);
        buffer = free.allocate(size, maxTimeToBlock);

        // Update the current time in case the buffer allocation blocked above.
        nowMs = time.milliseconds();
        synchronized (dq) {
            // Need to check if producer is closed again after grabbing the dequeue lock.
            if (closed)
                throw new KafkaException("Producer closed while send in progress");

            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq, nowMs);
            if (appendResult != null) {
                // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                return appendResult;
            }

            MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
            ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, nowMs);
            FutureRecordMetadata future = Objects.requireNonNull(batch.tryAppend(timestamp, key, value, headers,
                    callback, nowMs));

            dq.addLast(batch);
            incomplete.add(batch);

            // Don't deallocate this buffer in the finally block as it's being used in the record batch
            buffer = null;
            // 如果 dp.size() > 1 或者 batch.isFull() 就说明有 batch 可以发送
            return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true, false);
        }
    } finally {
        if (buffer != null)
            free.deallocate(buffer);
        appendsInProgress.decrementAndGet();
    }
}
```

### ready

获取已经准备好发送的 batch 对应的 leader (node)

### drain

返回

```
{
    // Map<Integer, List<ProducerBatch>>
    nodeId: [],
    nodeId: [],
    ...
}
```
