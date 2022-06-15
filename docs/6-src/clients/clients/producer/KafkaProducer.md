# Producer

1. 生产者维持一个 pool of buffer space 存储还没有发往 server 的 record
2. 同时有背后的 I/O thread 负责将 record 发往 server
3. `send` 方法是异步的，当调用时把 record 加入一个 buffer of pending record sends 并立即返回。这允许 producer 通过打包单独的 record 提供效率
4. `batch.size`: producer 为每个 partition 维护一个未发送 record 的 buffer，buffer 的大小由 `batch.size` 配置。增加配置可以让 record 打包增大，提高发送效率，但会增加内存使用，降低发送时效
5. `linger.ms`: 默认情况下，a buffer is available to send immediately even if there is additional unused space in the buffseer
7. `buffer.memory`: 控制 producer 可以使用的 buffer 的大小，如果 record 发送快于 producer 与 server 的交互，buffer 可用空间会被耗尽，`send` 调用会被阻塞。`max.block.ms` 配置阻塞的超时时间，如果超时，会抛出 `TimeoutException` 异常
8. 0.11 之后，Kafka 支持幂等(idempotent) 与事务(transacional)
    1. 为了支持幂等，`enable.idempotence` 必须被设置为 `true`，设置之后 `retries` 会被默认设为 `Integer.MAX_VALUE` 并且 `acks` 会默认设为 `all`
    2. 为了支持事务，`transactional.id` 必须被设置，设置之后，`enable.idempotence` 会自动开启。此外:
        1. topic which are included in transactions 应该 configured for durability，具体的: `replication.factor` 至少为 3，`min.insync.replicas` 应该设置为 2。
        2. 为了端到端的事务性保证，consumer 应该被设置为只读 committed messages

## 属性

``` java
private final Logger log;
private static final String JMX_PREFIX = "kafka.producer";
public static final String NETWORK_THREAD_PREFIX = "kafka-producer-network-thread";
public static final String PRODUCER_METRIC_GROUP_NAME = "producer-metrics";

private final String clientId;
// Visible for testing
final Metrics metrics;
private final Partitioner partitioner;
private final int maxRequestSize;
private final long totalMemorySize;
private final ProducerMetadata metadata;
private final RecordAccumulator accumulator;
private final Sender sender;
private final Thread ioThread;
private final CompressionType compressionType;
private final Sensor errors;
private final Time time;
private final Serializer<K> keySerializer;
private final Serializer<V> valueSerializer;
private final ProducerConfig producerConfig;
private final long maxBlockTimeMs;
private final ProducerInterceptors<K, V> interceptors;
private final ApiVersions apiVersions;
private final TransactionManager transactionManager;
```

## 构造方法

`KafkaProducer` 构造方法中，初始如下关键属性

* `this.partitioner` 分区算法
* `this.accumulator = new RecordAccumulator`
* `this.metadata = new ProducerMetadata`
    ``` java
    this.metadata = new ProducerMetadata(retryBackoffMs,
            config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG),
            config.getLong(ProducerConfig.METADATA_MAX_IDLE_CONFIG),
            logContext,
            clusterResourceListeners,
            Time.SYSTEM);
    this.metadata.bootstrap(addresses);
    ```
* `this.sender = newSender(logContext, kafkaClient, this.metadata)`

同时开启线程，运行 `sender`

``` java
this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
this.ioThread.start();
```

## send

``` java
/**
 * Asynchronously send a record to a topic and invoke the provided callback when the send has been acknowledged.
 * <p>
 * The send is asynchronous and this method will return immediately once the record has been stored in the buffer of
 * records waiting to be sent. This allows sending many records in parallel without blocking to wait for the
 * response after each one.
 * <p>
 * The result of the send is a {@link RecordMetadata} specifying the partition the record was sent to, the offset
 * it was assigned and the timestamp of the record. If
 * {@link org.apache.kafka.common.record.TimestampType#CREATE_TIME CreateTime} is used by the topic, the timestamp
 * will be the user provided timestamp or the record send time if the user did not specify a timestamp for the
 * record. If {@link org.apache.kafka.common.record.TimestampType#LOG_APPEND_TIME LogAppendTime} is used for the
 * topic, the timestamp will be the Kafka broker local time when the message is appended.
 * <p>
 * Since the send call is asynchronous it returns a {@link java.util.concurrent.Future Future} for the
 * {@link RecordMetadata} that will be assigned to this record. Invoking {@link java.util.concurrent.Future#get()
 * get()} on this future will block until the associated request completes and then return the metadata for the record
 * or throw any exception that occurred while sending the record.
 * <p>
 * If you want to simulate a simple blocking call you can call the <code>get()</code> method immediately:
 *
 * <pre>
 * {@code
 * byte[] key = "key".getBytes();
 * byte[] value = "value".getBytes();
 * ProducerRecord<byte[],byte[]> record = new ProducerRecord<byte[],byte[]>("my-topic", key, value)
 * producer.send(record).get();
 * }</pre>
 * <p>
 * Fully non-blocking usage can make use of the {@link Callback} parameter to provide a callback that
 * will be invoked when the request is complete.
 *
 * <pre>
 * {@code
 * ProducerRecord<byte[],byte[]> record = new ProducerRecord<byte[],byte[]>("the-topic", key, value);
 * producer.send(myRecord,
 *               new Callback() {
 *                   public void onCompletion(RecordMetadata metadata, Exception e) {
 *                       if(e != null) {
 *                          e.printStackTrace();
 *                       } else {
 *                          System.out.println("The offset of the record we just sent is: " + metadata.offset());
 *                       }
 *                   }
 *               });
 * }
 * </pre>
 *
 * Callbacks for records being sent to the same partition are guaranteed to execute in order. That is, in the
 * following example <code>callback1</code> is guaranteed to execute before <code>callback2</code>:
 *
 * <pre>
 * {@code
 * producer.send(new ProducerRecord<byte[],byte[]>(topic, partition, key1, value1), callback1);
 * producer.send(new ProducerRecord<byte[],byte[]>(topic, partition, key2, value2), callback2);
 * }
 * </pre>
 * <p>
 * When used as part of a transaction, it is not necessary to define a callback or check the result of the future
 * in order to detect errors from <code>send</code>. If any of the send calls failed with an irrecoverable error,
 * the final {@link #commitTransaction()} call will fail and throw the exception from the last failed send. When
 * this happens, your application should call {@link #abortTransaction()} to reset the state and continue to send
 * data.
 * </p>
 * <p>
 * Some transactional send errors cannot be resolved with a call to {@link #abortTransaction()}.  In particular,
 * if a transactional send finishes with a {@link ProducerFencedException}, a {@link org.apache.kafka.common.errors.OutOfOrderSequenceException},
 * a {@link org.apache.kafka.common.errors.UnsupportedVersionException}, or an
 * {@link org.apache.kafka.common.errors.AuthorizationException}, then the only option left is to call {@link #close()}.
 * Fatal errors cause the producer to enter a defunct state in which future API calls will continue to raise
 * the same underyling error wrapped in a new {@link KafkaException}.
 * </p>
 * <p>
 * It is a similar picture when idempotence is enabled, but no <code>transactional.id</code> has been configured.
 * In this case, {@link org.apache.kafka.common.errors.UnsupportedVersionException} and
 * {@link org.apache.kafka.common.errors.AuthorizationException} are considered fatal errors. However,
 * {@link ProducerFencedException} does not need to be handled. Additionally, it is possible to continue
 * sending after receiving an {@link org.apache.kafka.common.errors.OutOfOrderSequenceException}, but doing so
 * can result in out of order delivery of pending messages. To ensure proper ordering, you should close the
 * producer and create a new instance.
 * </p>
 * <p>
 * If the message format of the destination topic is not upgraded to 0.11.0.0, idempotent and transactional
 * produce requests will fail with an {@link org.apache.kafka.common.errors.UnsupportedForMessageFormatException}
 * error. If this is encountered during a transaction, it is possible to abort and continue. But note that future
 * sends to the same topic will continue receiving the same exception until the topic is upgraded.
 * </p>
 * <p>
 * Note that callbacks will generally execute in the I/O thread of the producer and so should be reasonably fast or
 * they will delay the sending of messages from other threads. If you want to execute blocking or computationally
 * expensive callbacks it is recommended to use your own {@link java.util.concurrent.Executor} in the callback body
 * to parallelize processing.
 *
 * @param record The record to send
 * @param callback A user-supplied callback to execute when the record has been acknowledged by the server (null
 *        indicates no callback)
 *
 * @throws AuthenticationException if authentication fails. See the exception for more details
 * @throws AuthorizationException fatal error indicating that the producer is not allowed to write
 * @throws IllegalStateException if a transactional.id has been configured and no transaction has been started, or
 *                               when send is invoked after producer has been closed.
 * @throws InterruptException If the thread is interrupted while blocked
 * @throws SerializationException If the key or value are not valid objects given the configured serializers
 * @throws KafkaException If a Kafka related error occurs that does not belong to the public API exceptions.
 */
@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // intercept the record, which can be potentially modified; this method does not throw exceptions
    ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
}
```

## doSend

``` java
/**
 * Implementation of asynchronously send a record to a topic.
 */
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        throwIfProducerClosed();
        // first make sure the metadata for the topic is available
        long nowMs = time.milliseconds();
        ClusterAndWaitTime clusterAndWaitTime;
        try {
            clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
        } catch (KafkaException e) {
            if (metadata.isClosed())
                throw new KafkaException("Producer closed while send in progress", e);
            throw e;
        }
        nowMs += clusterAndWaitTime.waitedOnMetadataMs;
        long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
        Cluster cluster = clusterAndWaitTime.cluster;
        // 3. 序列化 key, value
        byte[] serializedKey;
        try {
            serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in key.serializer", cce);
        }
        byte[] serializedValue;
        try {
            serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in value.serializer", cce);
        }
        // 4. 计算消息所处分区
        int partition = partition(record, serializedKey, serializedValue, cluster);
        tp = new TopicPartition(record.topic(), partition);

        setReadOnly(record.headers());
        Header[] headers = record.headers().toArray();

        // 5. 计算消息大小，确定不超过 `max.request.size`, `buffer.memory`
        int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                compressionType, serializedKey, serializedValue, headers);
        ensureValidRecordSize(serializedSize);
        // 6. 获取消息 timestamp，如果没有就用当前时间
        long timestamp = record.timestamp() == null ? nowMs : record.timestamp();
        if (log.isTraceEnabled()) {
            log.trace("Attempting to append record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
        }
        // producer callback will make sure to call both 'callback' and interceptor callback
        Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

        if (transactionManager != null && transactionManager.isTransactional()) {
            transactionManager.failIfNotReadyForSend();
        }
        // 7. 将消息追加到 accumulator
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);

        if (result.abortForNewBatch) {
            int prevPartition = partition;
            partitioner.onNewBatch(record.topic(), cluster, prevPartition);
            partition = partition(record, serializedKey, serializedValue, cluster);
            tp = new TopicPartition(record.topic(), partition);
            if (log.isTraceEnabled()) {
                log.trace("Retrying append due to new batch creation for topic {} partition {}. The old partition was {}", record.topic(), partition, prevPartition);
            }
            // producer callback will make sure to call both 'callback' and interceptor callback
            interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

            result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs, false, nowMs);
        }

        if (transactionManager != null && transactionManager.isTransactional())
            transactionManager.maybeAddPartitionToTransaction(tp);

        // 8. 如果 `batch` 满了或者新创建了 `batch`，唤醒 sender
        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
            this.sender.wakeup();
        }
        return result.future;
        // handling exceptions and record the errors;
        // for API exceptions return them in the future,
        // for other exceptions throw directly
    } catch (ApiException e) {
        log.debug("Exception occurred during message send:", e);
        if (callback != null)
            callback.onCompletion(null, e);
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        return new FutureFailure(e);
    } catch (InterruptedException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw new InterruptException(e);
    } catch (KafkaException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw e;
    } catch (Exception e) {
        // we notify interceptor about all exceptions, since onSend is called before anything else in this method
        this.interceptors.onSendError(record, tp, e);
        throw e;
    }
}
```
