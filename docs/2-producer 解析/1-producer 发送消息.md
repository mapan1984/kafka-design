# 2.1 producer 发送消息

## 代码示例

我们先看下 `KafkaProducer` 如何发送消息：

``` java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("acks", "all");
props.put("retries", 0);
props.put("linger.ms", 1);
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

Producer<String, String> producer = new KafkaProducer<>(props);
for (int i = 0; i < 100; i++) {
    producer.send(new ProducerRecord<String, String>("my-topic", Integer.toString(i), Integer.toString(i)));
}

producer.close();
```

这里只有 2 部分，构造 `KafkaProducer` 实例，调用 `send()` 函数。

## 构造 KafkaProducer

### KafkaProducer 实例

需要注意 `KafkaProducer` 中以下几个属性：

* `partitioner`: `Partitioner` 实例，对应分区算法
* `metadata`: `ProducerMetadata` 实例，包含 topic metadata
* `accumulator`: `RecordAccumulator` 实例，包含每个 topic-partition 发送数据的队列
* `sender`: `Sender` 实例，之后会在独立的线程中运行，负责数据的发送
* `ioThread`: `KafkaThread` 实例，独立的线程，运行 `sender`

### Sender 实例

需要注意 `Sender` 中以下几个属性：

* `client`: `KafkaClient` 实例(实际为`NetworkClient`)，进行实际的数据发送
* `accumulator`: `RecordAccumulator` 实例
* `metadata`: `ProducerMetadata` 实例，包含 topic metadata

### NetworkClient 实例

* `selector`: kafka 对 java.nio.channels.Selector 的包装
* `metadataUpdater`

KafkaProducer 初始化之后，创建的对象如下所示

![](/kafka-design/images/2-producer/kafka producer.jpeg)

`KafkaProducer` 构造方法中，初始如下关键属性

* `partitioner`
* `accumulator`
* `metadata`
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

## send() 做了什么？

`send()` 是异步的，调用会立即返回，record 会被存储到 the buffer of records 等待被发送。This allows sending many records in parallel without blocking to wait for the response after each one.

1. 使用 `ProducerInterceptors` 对 record 进行处理
2. 调用 `doSend`
    1. `throwIfProducerClosed`，检查 `sender` 是否存在且正在运行
    2. make sure the metadata for the topic is available，这里会一直尝试获取 metadata，直到超过 maxBlockTimeMs
        ``` java
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
        ```
    3. 序列化 key, value
    4. 计算消息所处分区
    5. 计算消息大小，确定不超过 `max.request.size`, `buffer.memory`
    6. 获取消息 timestamp，如果没有就用当前时间
    7. 将消息追加到 accumulator
        ``` java
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);
        ```
    8. 如果 `batch` 满了或者新创建了 `batch`，唤醒 sender
        ``` java
        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
            this.sender.wakeup();
        }
        ```

之后会由独立线程的 `Sender` 从 `RecordAccumulator` 中取得消息然后发送到 Kafka

## Sender 做了什么？

Sender 是作为独立线程运行，主要逻辑在 `run` 函数中:

``` java
// main loop, runs until close is called
while (running) {
    try {
        runOnce();
    } catch (Exception e) {
        log.error("Uncaught error in kafka producer I/O thread: ", e);
    }
}
```

`run` 循环运行 `runOnce`，`runOnce` 主要有 2 步（先不讨论事务性支持）：

- sendProducerData  // 将 record batch 转移到每个节点的生产请求列表中
    1. 获取 metadata
    2. 从 accumulator 的 batches 中取出可以发送的数据，
        1. 得到 batches 对应的 leader 节点列表 （`RecordAccumulator.ReadyCheckResult result.readyNodes`）
        2. 得到在 metadata 里找不到 leader 的 topic 列表（`RecordAccumulator.ReadyCheckResult result.unknownLeaderTopics`）
    3. 如果有任何 partition 的 leader 找不到，更新 metadata
    4. 遍历 leader node 列表(`result.readyNodes`)
        1. 判断 leader node 是否已经连接并可以发送数据
            1. 如果不是，则 **开始连接**，并在 `readyNodes` 中移除这个 leader node
    5. 获取每个 leader 对应的可以发送的 `ProducerBatch` 列表
    6. 在 Sender 的 inFlightBatches 里记录 topicPartition 对应的 batch 列表
    7. 如果需要保证消息的顺序，将 topicPartition 加入到 accumulator 的 muted 记录
    8. `sendProduceRequests(batches, now)`
        ``` java
        /**
         * Transfer the record batches into a list of produce requests on a per-node basis
         */
        private void sendProduceRequests(Map<Integer, List<ProducerBatch>> collated, long now) {
            for (Map.Entry<Integer, List<ProducerBatch>> entry : collated.entrySet())
                sendProduceRequest(now, entry.getKey(), acks, requestTimeoutMs, entry.getValue());
        }
        ```
    9. `sendProduceRequest` 里构造 `ProduceRequest.Builder`，构造 `ClientRequest`，并调用 `NetworkClient` 的 `send()` 函数发送
- client.poll  // `NetworkClient.poll` Do actual reads and writes to sockets

这里有主要的几点：

1. **何时与 broker 建立连接**：
    1. 获取 `readyNodes` 之后，会遍历 `readyNodes` 检查检查 node 是否已经连接，这里调用 `NetworkClient`.`ready()`
    2. `NetworkClient`.`ready()` 中如果判断 node 已连接，会直接返回 `true`，如果没有连接，则进行连接，并返回 `false`
        ``` java
        @Override
        public boolean ready(Node node, long now) {
            if (node.isEmpty())
                throw new IllegalArgumentException("Cannot connect to empty node " + node);

            if (isReady(node, now))
                return true;

            if (connectionStates.canConnect(node.idString(), now))
                // if we are interested in sending to a node and we don't have a connection to it, initiate one
                initiateConnect(node, now);

            return false;
        }
        ```
    3. `NetworkClient`.`initiateConnect()` 中通过 `Selector`.`connect` 与 node 进行连接
        ``` java
        private void initiateConnect(Node node, long now) {
            String nodeConnectionId = node.idString();
            try {
                connectionStates.connecting(nodeConnectionId, now, node.host(), clientDnsLookup);
                InetAddress address = connectionStates.currentAddress(nodeConnectionId);
                log.debug("Initiating connection to node {} using address {}", node, address);
                selector.connect(nodeConnectionId,
                        new InetSocketAddress(address, node.port()),
                        this.socketSendBuffer,
                        this.socketReceiveBuffer);
            } catch (IOException e) {
                log.warn("Error connecting to node {}", node, e);
                // Attempt failed, we'll try again after the backoff
                connectionStates.disconnected(nodeConnectionId, now);
                // Notify metadata updater of the connection failure
                metadataUpdater.handleServerDisconnect(now, nodeConnectionId, Optional.empty());
            }
        }
        ```
    4. `Selector`.`connect`:
        1. 创建 `SocketChannel`，进行连接 node 操作
        2. 注册 `SocketChannel` 到 `Selector`.`nioSelector`
        3. 构建 `KafkaChannel`（通过 (Ssl/Sasl/Plaintext)ChannelBuilder 构建）
        4. 将 `KafakChannel` 记录到 `Selector`.`channels` 中


> kafka 发送消息时以节点组织可发送的消息，将发往同一个节点的不同 topicPartion 的数据放在一个请求中发送

## NetworkClient 是如何发送数据的？

## Selector 的 poll

- `poll`
    - `pollSelectionKeys(Set<SelectionKey> selectionKeys, boolean isImmediatelyConnected, long currentTimeNanos)`
        - `attempRead(KafkaChannel channel)`: 
            1. 调用 `channel.read()`，返回读取的 byte 数，
                - `KafkaChannel`.`read()`
                    - `KafkaChannel`.`receive()` 通过调用自身所持有的 `NetworkReceive` 的 `readFrom()` 函数进行数据读取
            2. 通过 `channel.maybeCompleteReceive()` 判断是否收到完整的 `NetworkReceive`，如果是，将 receive 加入到自身的 `completedReceives`
        - `attemptWrite(SelectionKey key, KafkaChannel channel, long nowNanos)`
            - `write(KafkaChannel channel)`: 
                1. 调用 `channel.write()`，返回发送 byte 数，
                    - `KafkaChannel`.`write()` 通过调用自身所持有 `NetworkSend` 的 `writeTo()` 函数进行数据发送
                        - `java.nio.channels.GatheringByteChannel`.`write()`
                2. 通过 `channel.maybeCompleteSend()` 判断 `NetworkSend` 是否发送完成，如果是，将 send 加入到自身的 `completedSends`

这里有主要的几点：

1. **何时进行 ssl, sasl 握手**：在 `Sender`.`run()` 中会检查要发送数据对应的 leader 是否连接，如果没有则进行连接，在 `Selector`.`pollSelectionKeys()` 中，如果 key 对应的 channel 已连接但还未 ready，则调用 `KafkaChannel`.`prepare()` 进行 ssl 握手和 sasl 握手

## 参考

https://blog.csdn.net/chunlongyu/article/details/52651960
