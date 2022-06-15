# 1.4.1 replica manager 启动

isr shrik 和 isr expand 都是 leader 发起的。

leader 有定时任务检查 isr 每个 replica 是否发生落后，replica fetch 请求到来时，检查是否要将该 replica 加入 isr。

zk 上该 partition 的 state 信息也由 leader 更新。

replica manager 在 `kafkaServer.startup()` 中初始化并启动：

``` scala
  /**
   * Start up API for bringing up a single instance of the Kafka server.
   * Instantiates the LogManager, the SocketServer and the request handlers - KafkaRequestHandlers
   */
  def startup(): Unit = {
    try {
      info("starting")

      if (isShuttingDown.get)
        throw new IllegalStateException("Kafka server is still shutting down, cannot re-start!")

      if (startupComplete.get)
        return

      val canStartup = isStartingUp.compareAndSet(false, true)
      if (canStartup) {

        // ...

        /* start replica manager */
        brokerToControllerChannelManager = new BrokerToControllerChannelManagerImpl(metadataCache, time, metrics, config, threadNamePrefix)
        replicaManager = createReplicaManager(isShuttingDown)
        replicaManager.startup()
        brokerToControllerChannelManager.start()

        // ...
```

``` scala
  protected def createReplicaManager(isShuttingDown: AtomicBoolean): ReplicaManager = {
    val alterIsrManager = new AlterIsrManagerImpl(brokerToControllerChannelManager, kafkaScheduler,
      time, config.brokerId, () => kafkaController.brokerEpoch)
    new ReplicaManager(config, metrics, time, zkClient, kafkaScheduler, logManager, isShuttingDown, quotaManagers,
      brokerTopicStats, metadataCache, logDirFailureChannel, alterIsrManager)
  }
```

`ReplicaManager.startup()` 方法会启动以下定时任务：
1. isr-expiration：定时检查 isr 列表中是否有 replica 需要被移除，这里会对 isr 变化的 topicPartition 进行记录，分为 2 种情况：
    1. 如果 api version `>=` KAFKA_2_7_IV2，记录在 alterIsrManager 的 unsentIsrUpdates 中
    2. 否则记录在自身的 isrChangeSet 中
2. isr 的变化需要通知到 controller，上一步的定时任务只是记录，这里会根据集群支持的 API version 分别通过 AlterIsrRequest 请求或者 zookeeper 向 controller 进行通知:
    1. send-alter-isr: 如果 api version `>=` KAFKA_2_7_IV2，启动 alterIsrManager，alterIsrManager 同样启动定时任务，定期向 controller 发送未完成的 AlterIsrRequest 请求
    2. isr-change-propagation: 否则定时检查 isr 是否需要变动，创建 `/isr_change_notification/isr_change_<>` zookeeper 节点，触发 controller 动作
3. shutdown-idle-replica-alter-log-dirs-thread

``` scala
  def startup(): Unit = {
    // start ISR expiration thread
    // A follower can lag behind leader for up to config.replicaLagTimeMaxMs x 1.5 before it is removed from ISR
    scheduler.schedule("isr-expiration", maybeShrinkIsr _, period = config.replicaLagTimeMaxMs / 2, unit = TimeUnit.MILLISECONDS)
    // If using AlterIsr, we don't need the znode ISR propagation
    if (!config.interBrokerProtocolVersion.isAlterIsrSupported) {
      scheduler.schedule("isr-change-propagation", maybePropagateIsrChanges _,
        period = isrChangeNotificationConfig.checkIntervalMs, unit = TimeUnit.MILLISECONDS)
    } else {
      alterIsrManager.start()
    }
    scheduler.schedule("shutdown-idle-replica-alter-log-dirs-thread", shutdownIdleReplicaAlterLogDirsThread _, period = 10000L, unit = TimeUnit.MILLISECONDS)

    // If inter-broker protocol (IBP) < 1.0, the controller will send LeaderAndIsrRequest V0 which does not include isNew field.
    // In this case, the broker receiving the request cannot determine whether it is safe to create a partition if a log directory has failed.
    // Thus, we choose to halt the broker on any log diretory failure if IBP < 1.0
    val haltBrokerOnFailure = config.interBrokerProtocolVersion < KAFKA_1_0_IV0
    logDirFailureHandler = new LogDirFailureHandler("LogDirFailureHandler", haltBrokerOnFailure)
    logDirFailureHandler.start()
  }
```

## isr shrink

定时任务 maybeShrinkIsr 会遍历 topic partition，对 partition 实例调用其 `maybeShrinkIsr()` 方法

``` scala
  private def maybeShrinkIsr(): Unit = {
    trace("Evaluating ISR list of partitions to see which replicas can be removed from the ISR")

    // Shrink ISRs for non offline partitions
    allPartitions.keys.foreach { topicPartition =>
      nonOfflinePartition(topicPartition).foreach(_.maybeShrinkIsr())
    }
  }
```

`Partition.maybeShrinkIsr()` 定义如下

``` scala
  def maybeShrinkIsr(): Unit = {
    val needsIsrUpdate = !isrState.isInflight && inReadLock(leaderIsrUpdateLock) {
      needsShrinkIsr()
    }
    val leaderHWIncremented = needsIsrUpdate && inWriteLock(leaderIsrUpdateLock) {
      leaderLogIfLocal.exists { leaderLog =>
        val outOfSyncReplicaIds = getOutOfSyncReplicas(replicaLagTimeMaxMs)
        if (outOfSyncReplicaIds.nonEmpty) {
          val outOfSyncReplicaLog = outOfSyncReplicaIds.map { replicaId =>
            s"(brokerId: $replicaId, endOffset: ${getReplicaOrException(replicaId).logEndOffset})"
          }.mkString(" ")
          val newIsrLog = (isrState.isr -- outOfSyncReplicaIds).mkString(",")
          info(s"Shrinking ISR from ${isrState.isr.mkString(",")} to $newIsrLog. " +
               s"Leader: (highWatermark: ${leaderLog.highWatermark}, endOffset: ${leaderLog.logEndOffset}). " +
               s"Out of sync replicas: $outOfSyncReplicaLog.")

          shrinkIsr(outOfSyncReplicaIds)

          // we may need to increment high watermark since ISR could be down to 1
          maybeIncrementLeaderHW(leaderLog)
        } else {
          false
        }
      }
    }

    // some delayed operations may be unblocked after HW changed
    if (leaderHWIncremented)
      tryCompleteDelayedRequests()
  }
```

`Partition.shrinkIsr()`

``` scala
  private[cluster] def shrinkIsr(outOfSyncReplicas: Set[Int]): Unit = {
    if (useAlterIsr) {
      shrinkIsrWithAlterIsr(outOfSyncReplicas)
    } else {
      shrinkIsrWithZk(isrState.isr -- outOfSyncReplicas)
    }
  }

  private def shrinkIsrWithAlterIsr(outOfSyncReplicas: Set[Int]): Unit = {
    // This is called from maybeShrinkIsr which holds the ISR write lock
    if (!isrState.isInflight) {
      // When shrinking the ISR, we cannot assume that the update will succeed as this could erroneously advance the HW
      // We update pendingInSyncReplicaIds here simply to prevent any further ISR updates from occurring until we get
      // the next LeaderAndIsr
      sendAlterIsrRequest(PendingShrinkIsr(isrState.isr, outOfSyncReplicas))
    } else {
      trace(s"ISR update in-flight, not removing out-of-sync replicas $outOfSyncReplicas")
    }
  }

  private def shrinkIsrWithZk(newIsr: Set[Int]): Unit = {
    val newLeaderAndIsr = new LeaderAndIsr(localBrokerId, leaderEpoch, newIsr.toList, zkVersion)
    val zkVersionOpt = stateStore.shrinkIsr(controllerEpoch, newLeaderAndIsr)
    if (zkVersionOpt.isDefined) {
      isrChangeListener.markShrink()
    }
    maybeUpdateIsrAndVersionWithZk(newIsr, zkVersionOpt)
  }
```

这里也分 2 种情况：
1. 如果 api version `>=` KAFKA_2_7_IV2，即 `useAlterIsr`，向 alterIsrManager 增加 AlterIsrRequest 记录
2. 否则，在 ReplicaManager 的 isrChangeSet 中记录 isr 变化的 topicPartition


## isr expand / fetchMessages

topic partition leader 所在节点，需要记录每个 replica 的 logEndOffset，每个 replica 都要记录自身的 HW

``` scala
  /**
   * Fetch messages from a replica, and wait until enough data can be fetched and return;
   * the callback function will be triggered either when timeout or required fetch info is satisfied.
   * Consumers may fetch from any replica, but followers can only fetch from the leader.
   */
  def fetchMessages(timeout: Long,
                    replicaId: Int,
                    fetchMinBytes: Int,
                    fetchMaxBytes: Int,
                    hardMaxBytesLimit: Boolean,
                    fetchInfos: Seq[(TopicPartition, PartitionData)],
                    quota: ReplicaQuota,
                    responseCallback: Seq[(TopicPartition, FetchPartitionData)] => Unit,
                    isolationLevel: IsolationLevel,
                    clientMetadata: Option[ClientMetadata]): Unit = {
    val isFromFollower = Request.isValidBrokerId(replicaId)
    val isFromConsumer = !(isFromFollower || replicaId == Request.FutureLocalReplicaId)
    val fetchIsolation = if (!isFromConsumer)
      FetchLogEnd
    else if (isolationLevel == IsolationLevel.READ_COMMITTED)
      FetchTxnCommitted
    else
      FetchHighWatermark

    // Restrict fetching to leader if request is from follower or from a client with older version (no ClientMetadata)
    val fetchOnlyFromLeader = isFromFollower || (isFromConsumer && clientMetadata.isEmpty)
    def readFromLog(): Seq[(TopicPartition, LogReadResult)] = {
      val result = readFromLocalLog(
        replicaId = replicaId,
        fetchOnlyFromLeader = fetchOnlyFromLeader,
        fetchIsolation = fetchIsolation,
        fetchMaxBytes = fetchMaxBytes,
        hardMaxBytesLimit = hardMaxBytesLimit,
        readPartitionInfo = fetchInfos,
        quota = quota,
        clientMetadata = clientMetadata)
      if (isFromFollower) updateFollowerFetchState(replicaId, result)
      else result
    }

    val logReadResults = readFromLog()

    // check if this fetch request can be satisfied right away
    var bytesReadable: Long = 0
    var errorReadingData = false
    var hasDivergingEpoch = false
    val logReadResultMap = new mutable.HashMap[TopicPartition, LogReadResult]
    logReadResults.foreach { case (topicPartition, logReadResult) =>
      brokerTopicStats.topicStats(topicPartition.topic).totalFetchRequestRate.mark()
      brokerTopicStats.allTopicsStats.totalFetchRequestRate.mark()

      if (logReadResult.error != Errors.NONE)
        errorReadingData = true
      if (logReadResult.divergingEpoch.nonEmpty)
        hasDivergingEpoch = true
      bytesReadable = bytesReadable + logReadResult.info.records.sizeInBytes
      logReadResultMap.put(topicPartition, logReadResult)
    }

    // respond immediately if 1) fetch request does not want to wait
    //                        2) fetch request does not require any data
    //                        3) has enough data to respond
    //                        4) some error happens while reading data
    //                        5) we found a diverging epoch
    if (timeout <= 0 || fetchInfos.isEmpty || bytesReadable >= fetchMinBytes || errorReadingData || hasDivergingEpoch) {
      val fetchPartitionData = logReadResults.map { case (tp, result) =>
        val isReassignmentFetch = isFromFollower && isAddingReplica(tp, replicaId)
        tp -> FetchPartitionData(
          result.error,
          result.highWatermark,
          result.leaderLogStartOffset,
          result.info.records,
          result.divergingEpoch,
          result.lastStableOffset,
          result.info.abortedTransactions,
          result.preferredReadReplica,
          isReassignmentFetch)
      }
      responseCallback(fetchPartitionData)
    } else {
      // construct the fetch results from the read results
      val fetchPartitionStatus = new mutable.ArrayBuffer[(TopicPartition, FetchPartitionStatus)]
      fetchInfos.foreach { case (topicPartition, partitionData) =>
        logReadResultMap.get(topicPartition).foreach(logReadResult => {
          val logOffsetMetadata = logReadResult.info.fetchOffsetMetadata
          fetchPartitionStatus += (topicPartition -> FetchPartitionStatus(logOffsetMetadata, partitionData))
        })
      }
      val fetchMetadata: SFetchMetadata = SFetchMetadata(fetchMinBytes, fetchMaxBytes, hardMaxBytesLimit,
        fetchOnlyFromLeader, fetchIsolation, isFromFollower, replicaId, fetchPartitionStatus)
      val delayedFetch = new DelayedFetch(timeout, fetchMetadata, this, quota, clientMetadata,
        responseCallback)

      // create a list of (topic, partition) pairs to use as keys for this delayed fetch operation
      val delayedFetchKeys = fetchPartitionStatus.map { case (tp, _) => TopicPartitionOperationKey(tp) }

      // try to complete the request immediately, otherwise put it into the purgatory;
      // this is because while the delayed fetch operation is being created, new requests
      // may arrive and hence make this operation completable.
      delayedFetchPurgatory.tryCompleteElseWatch(delayedFetch, delayedFetchKeys)
    }
  }
```


updateFollowerFetchState


``` scala
  /**
   * Update the follower's fetch state on the leader based on the last fetch request and update `readResult`.
   * If the follower replica is not recognized to be one of the assigned replicas, do not update
   * `readResult` so that log start/end offset and high watermark is consistent with
   * records in fetch response. Log start/end offset and high watermark may change not only due to
   * this fetch request, e.g., rolling new log segment and removing old log segment may move log
   * start offset further than the last offset in the fetched records. The followers will get the
   * updated leader's state in the next fetch response.
   */
  private def updateFollowerFetchState(followerId: Int,
                                       readResults: Seq[(TopicPartition, LogReadResult)]): Seq[(TopicPartition, LogReadResult)] = {
    readResults.map { case (topicPartition, readResult) =>
      val updatedReadResult = if (readResult.error != Errors.NONE) {
        debug(s"Skipping update of fetch state for follower $followerId since the " +
          s"log read returned error ${readResult.error}")
        readResult
      } else {
        nonOfflinePartition(topicPartition) match {
          case Some(partition) =>
            if (partition.updateFollowerFetchState(followerId,
              followerFetchOffsetMetadata = readResult.info.fetchOffsetMetadata,
              followerStartOffset = readResult.followerLogStartOffset,
              followerFetchTimeMs = readResult.fetchTimeMs,
              leaderEndOffset = readResult.leaderLogEndOffset)) {
              readResult
            } else {
              warn(s"Leader $localBrokerId failed to record follower $followerId's position " +
                s"${readResult.info.fetchOffsetMetadata.messageOffset}, and last sent HW since the replica " +
                s"is not recognized to be one of the assigned replicas ${partition.assignmentState.replicas.mkString(",")} " +
                s"for partition $topicPartition. Empty records will be returned for this partition.")
              readResult.withEmptyFetchInfo
            }
          case None =>
            warn(s"While recording the replica LEO, the partition $topicPartition hasn't been created.")
            readResult
        }
      }
      topicPartition -> updatedReadResult
    }
  }
```
