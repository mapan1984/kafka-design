# replica fetcher manager

`ReplicaManager` 会创建一个 `ReplicaFetcherManager` 实例管理 `ReplicaFetcherThread` 线程。

`ReplicaFetcherThread` 线程按拉取源 broker id 分类，对每个源 broker 可以创建 numFetchersPerBroker 个 Fetcher 线程。

- leader broker 1
	- fetcher 1 (topic partition 1, topic partition 2)
	- fetcher 2 ()
- leader broker 2
	- fetcher 1 ()
	- fetcher 2 ()

``` scala
  private[server] def getFetcherId(topicPartition: TopicPartition): Int = {
    lock synchronized {
      Utils.abs(31 * topicPartition.topic.hashCode() + topicPartition.partition) % numFetchersPerBroker
    }
  }
```

``` scala
  def addFetcherForPartitions(partitionAndOffsets: Map[TopicPartition, InitialFetchState]): Unit = {
    lock synchronized {
      val partitionsPerFetcher = partitionAndOffsets.groupBy { case (topicPartition, brokerAndInitialFetchOffset) =>
        BrokerAndFetcherId(brokerAndInitialFetchOffset.leader, getFetcherId(topicPartition))
      }

      def addAndStartFetcherThread(brokerAndFetcherId: BrokerAndFetcherId,
                                   brokerIdAndFetcherId: BrokerIdAndFetcherId): T = {
        val fetcherThread = createFetcherThread(brokerAndFetcherId.fetcherId, brokerAndFetcherId.broker)
        fetcherThreadMap.put(brokerIdAndFetcherId, fetcherThread)
        fetcherThread.start()
        fetcherThread
      }

      for ((brokerAndFetcherId, initialFetchOffsets) <- partitionsPerFetcher) {
        val brokerIdAndFetcherId = BrokerIdAndFetcherId(brokerAndFetcherId.broker.id, brokerAndFetcherId.fetcherId)
        val fetcherThread = fetcherThreadMap.get(brokerIdAndFetcherId) match {
          case Some(currentFetcherThread) if currentFetcherThread.sourceBroker == brokerAndFetcherId.broker =>
            // reuse the fetcher thread
            currentFetcherThread
          case Some(f) =>
            f.shutdown()
            addAndStartFetcherThread(brokerAndFetcherId, brokerIdAndFetcherId)
          case None =>
            addAndStartFetcherThread(brokerAndFetcherId, brokerIdAndFetcherId)
        }

        val initialOffsetAndEpochs = initialFetchOffsets.map { case (tp, brokerAndInitOffset) =>
          tp -> OffsetAndEpoch(brokerAndInitOffset.initOffset, brokerAndInitOffset.currentLeaderEpoch)
        }

        addPartitionsToFetcherThread(fetcherThread, initialOffsetAndEpochs)
      }
    }
  }
```

## ReplicaFetcherThread

`processPartitionData()` 将数据写入实际的 log 文件，并且尝试更新自身的 high water mark 与 log start offset。

### 限流实现

#### ReplicaQuota - ReplicationQuotaManager

``` scala
trait ReplicaQuota {
  // 记录流量
  def record(value: Long): Unit

  // topicPartition 是否限流
  def isThrottled(topicPartition: TopicPartition): Boolean

  // 设置的限流是否耗尽
  def isQuotaExceeded: Boolean
}
```

#### follower 限流实现

`ReplicaFetcherManager` 通过 `createFetcherThread()` 创建 `ReplicaFetcherThread` 实例，会向其传递一个 `quotaManager: ReplicationQuotaManager`

在 `processPartitionData()` 方法中，记录副本同步数据大小

``` scala
// Traffic from both in-sync and out of sync replicas are accounted for in replication quota to ensure total replication
// traffic doesn't exceed quota.
if (quota.isThrottled(topicPartition))
  quota.record(records.sizeInBytes)
```

在 `buildFetch()` 方法中，在流量超过限制后过滤设置限流的 topicPartition

``` scala
val builder = fetchSessionHandler.newBuilder(partitionMap.size, false)
partitionMap.forKeyValue { (topicPartition, fetchState) =>
  // We will not include a replica in the fetch request if it should be throttled.
  if (fetchState.isReadyForFetch && !shouldFollowerThrottle(quota, fetchState, topicPartition)) {
    try {
      val logStartOffset = this.logStartOffset(topicPartition)
      builder.add(topicPartition, new FetchRequest.PartitionData(
        fetchState.fetchOffset, logStartOffset, fetchSize, Optional.of(fetchState.currentLeaderEpoch)))
    } catch {
      case _: KafkaStorageException =>
        // The replica has already been marked offline due to log directory failure and the original failure should have already been logged.
        // This partition should be removed from ReplicaFetcherThread soon by ReplicaManager.handleLogDirFailure()
        partitionsWithError += topicPartition
    }
  }
}

val fetchData = builder.build()
```

#### leader 限流实现

leader 限流在 `KafkaApis` 的 `handleFetchRequest()` 方法中

记录拉取消息大小

``` scala
if (fetchRequest.isFromFollower) {
  // We've already evaluated against the quota and are good to go. Just need to record it now.
  unconvertedFetchResponse = fetchContext.updateAndGenerateResponseData(partitions)
  val responseSize = sizeOfThrottledPartitions(versionId, unconvertedFetchResponse, quotas.leader)
  quotas.leader.record(responseSize)
  trace(s"Sending Fetch response with partitions.size=${unconvertedFetchResponse.responseData.size}, " +
    s"metadata=${unconvertedFetchResponse.sessionId}")
  sendResponseExemptThrottle(request, createResponse(0), Some(updateConversionStats))
```


