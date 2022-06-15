# 1.3.3 controller

在 controller 启动过程中，在 `KafkaController.onControllerFailover()` 方法中向 zkClient 注册了 `PartitionReassignmentHandler`

`PartitionReassignmentHandler` 如下：

``` scala
class PartitionReassignmentHandler(eventManager: ControllerEventManager) extends ZNodeChangeHandler {
  override val path: String = ReassignPartitionsZNode.path

  // Note that the event is also enqueued when the znode is deleted, but we do it explicitly instead of relying on
  // handleDeletion(). This approach is more robust as it doesn't depend on the watcher being re-registered after
  // it's consumed during data changes (we ensure re-registration when the znode is deleted).
  override def handleCreation(): Unit = eventManager.put(ZkPartitionReassignment)
}
```

这里实际上是监听 zookeeper `/admin/reassign_partitions` 节点，如果有新节点创建，则向 eventManager 事件队列 queue 增加 `ZkPartitionReassignment`

eventManager 会启动单独的线程不断从事件队列中取出事件，通过 processor 进行处理

eventManager 的 processor 是 KafkaController 的实例，因此可以在 `KafkaController.process()` 方法里找到对 `ZkPartitionReassignment` 的处理

``` scala
  override def process(event: ControllerEvent): Unit = {
    try {
      event match {
        case event: MockEvent =>
          // Used only in test cases
          event.process()

        // ...

        case ApiPartitionReassignment(reassignments, callback) =>
          processApiPartitionReassignment(reassignments, callback)

        case ZkPartitionReassignment =>
          processZkPartitionReassignment()

        case ListPartitionReassignments(partitions, callback) =>
          processListPartitionReassignments(partitions, callback)

        case PartitionReassignmentIsrChange(partition) =>
          processPartitionReassignmentIsrChange(partition)

        // ...

      }
    } catch {
      case e: ControllerMovedException =>
        info(s"Controller moved to another broker when processing $event.", e)
        maybeResign()
      case e: Throwable =>
        error(s"Error processing event $event", e)
    } finally {
      updateMetrics()
    }
  }
```

对 `ZkPartitionReassignment` 通过 `processZkPartitionReassignment()` 进行处理：

``` scala
  private def processZkPartitionReassignment(): Set[TopicPartition] = {
    // We need to register the watcher if the path doesn't exist in order to detect future
    // reassignments and we get the `path exists` check for free
    if (isActive && zkClient.registerZNodeChangeHandlerAndCheckExistence(partitionReassignmentHandler)) {
      val reassignmentResults = mutable.Map.empty[TopicPartition, ApiError]
      val partitionsToReassign = mutable.Map.empty[TopicPartition, ReplicaAssignment]

      // 这里组合了现在的 replica 和 targetReplica
      zkClient.getPartitionReassignment.forKeyValue { (tp, targetReplicas) =>
        maybeBuildReassignment(tp, Some(targetReplicas)) match {
          case Some(context) => partitionsToReassign.put(tp, context)
          case None => reassignmentResults.put(tp, new ApiError(Errors.NO_REASSIGNMENT_IN_PROGRESS))
        }
      }

      // 触发 partition reassignment
      reassignmentResults ++= maybeTriggerPartitionReassignment(partitionsToReassign)
      val (partitionsReassigned, partitionsFailed) = reassignmentResults.partition(_._2.error == Errors.NONE)
      if (partitionsFailed.nonEmpty) {
        warn(s"Failed reassignment through zk with the following errors: $partitionsFailed")
        maybeRemoveFromZkReassignment((tp, _) => partitionsFailed.contains(tp))
      }
      partitionsReassigned.keySet
    } else {
      Set.empty
    }
  }
```

``` scala
/**
 * Trigger a partition reassignment provided that the topic exists and is not being deleted.
 *
 * This is called when a reassignment is initially received either through Zookeeper or through the
 * AlterPartitionReassignments API
 *
 * The `partitionsBeingReassigned` field in the controller context will be updated by this
 * call after the reassignment completes validation and is successfully stored in the topic
 * assignment zNode.
 *
 * @param reassignments The reassignments to begin processing
 * @return A map of any errors in the reassignment. If the error is NONE for a given partition,
 *         then the reassignment was submitted successfully.
 */
private def maybeTriggerPartitionReassignment(reassignments: Map[TopicPartition, ReplicaAssignment]): Map[TopicPartition, ApiError] = {
  reassignments.map { case (tp, reassignment) =>
    val topic = tp.topic

    val apiError = if (topicDeletionManager.isTopicQueuedUpForDeletion(topic)) {
      info(s"Skipping reassignment of $tp since the topic is currently being deleted")
      new ApiError(Errors.UNKNOWN_TOPIC_OR_PARTITION, "The partition does not exist.")
    } else {
      val assignedReplicas = controllerContext.partitionReplicaAssignment(tp)
      if (assignedReplicas.nonEmpty) {
        try {
          onPartitionReassignment(tp, reassignment)
          ApiError.NONE
        } catch {
          case e: ControllerMovedException =>
            info(s"Failed completing reassignment of partition $tp because controller has moved to another broker")
            throw e
          case e: Throwable =>
            error(s"Error completing reassignment of partition $tp", e)
            new ApiError(Errors.UNKNOWN_SERVER_ERROR)
        }
      } else {
        new ApiError(Errors.UNKNOWN_TOPIC_OR_PARTITION, "The partition does not exist.")
      }
    }

    tp -> apiError
  }
}
```

定义以下概念：
- RS: 当前副本集合
- ORS: 原始副本集合
- TRS: 目标副本集合
- AR: 在这次重分配中要增加的副本
- RR: 在这次重分配中要移除的副本

假设一个 topic 当前副本集为 1,2,3 ，则其 RS 和 ORS 为 1,2,3

如果要将其重分配为 4,5,6 ，则其 TRS 为 4,5,6 , AR 为 4,5,6 , RR 为 1,2,3

| RS          | AR    | RR    | leader | isr         | step          |
|-------------|-------|-------|--------|-------------|---------------|
| 1,2,3       |       |       | 1      | 1,2,3       | initial state |
| 4,5,6,1,2,3 | 4,5,6 | 1,2,3 | 1      | 1,2,3       | step A2       |
| 4,5,6,1,2,3 | 4,5,6 | 1,2,3 | 1      | 1,2,3,4,5,6 | phase B       |
| 4,5,6,1,2,3 | 4,5,6 | 1,2,3 | 4      | 1,2,3,4,5,6 | step B3       |
| 4,5,6,1,2,3 | 4,5,6 | 1,2,3 | 4      | 4,5,6       | step B4       |
| 4,5,6       |       |       | 4      | 4,5,6       | step B6       |

将整个重分配过程分为 3 个阶段，每个阶段有自己的步骤
1. Phase U (Assignment update)
    1. U1: 更新 Zk 使 RS = ORS + TRS, AR = TRS - ORS, RR = ORS - TRS
    2. U2: 更新 memory 使 RS = ORS + TRS, AR = TRS - ORS, RR = ORS - TRS
    3. U3: 如果正在取消当前的重分配或者替换当前的重分配，向所有不在新分配的 TRS 的 AR 发送 `StopReplica` 请求
2. Phase A (when TRS != ISR)
    1. A1: bump the leader epoch for the partition and send LeaderAndIsr updates to RS
    2. A2: 通过将 AR 中的 replicas 置为 `NewReplica` 状态启动新的 replicas
3. Phase B (when TRS == ISR)
    1. B1: 将所有 AR 中的 replicas 置为 `OnlineReplica` 状态
    2. B2: 更新 memory 使 RS = TRS, AR = [], RR = []
    3. B3: 发送表示 RS = TRS 的 `LeaderAndIsr` 请求。This will prevent the leader from adding any replica in TRS - ORS back in the isr. 如果当前 leader 不在 TRS 里或者不是存活的，we move the leader to a new replica in TRS. We may send the LeaderAndIsr to more than the TRS replicas due to the way the partition state machine works (it reads replicas from ZK)
    4. B4: 将所有 RR 中的 replicas 置为 `OfflineReplica` 状态。As part of `OfflineReplica` state change, we shrink the isr to remove RR in ZooKeeper and send a `LeaderAndIsr` ONLY to the Leader to notify it of the shrunk isr. After that, we send a `StopReplica (delete = false)` to the replicas in RR.
    5. B5: 将所有 RR 中的 replicas 置为 `NonExistentReplica` 状态。This will send a `StopReplica (delete = true)` to the replicas in RR to physically delete the replicas on disk.
    6. B6: 更新 Zk 使 RS = TRS, AR = [], RR = []
    7. B7: 移除 ISR reassign listener，更新 Zk 的 `/admin/reassign_partitions` 移除这个 partition
    8. B8: 经过 leader 选举，replicas 和 isr 信息有变更，向所有 broker 发 metadata request

``` scala
/**
 * This callback is invoked:
 * 1. By the AlterPartitionReassignments API
 * 2. By the reassigned partitions listener which is triggered when the /admin/reassign/partitions znode is created
 * 3. When an ongoing reassignment finishes - this is detected by a change in the partition's ISR znode
 * 4. Whenever a new broker comes up which is part of an ongoing reassignment
 * 5. On controller startup/failover
 *
 * Reassigning replicas for a partition goes through a few steps listed in the code.
 * RS = current assigned replica set
 * ORS = Original replica set for partition
 * TRS = Reassigned (target) replica set
 * AR = The replicas we are adding as part of this reassignment
 * RR = The replicas we are removing as part of this reassignment
 *
 * A reassignment may have up to three phases, each with its own steps:

 * Phase U (Assignment update): Regardless of the trigger, the first step is in the reassignment process
 * is to update the existing assignment state. We always update the state in Zookeeper before
 * we update memory so that it can be resumed upon controller fail-over.
 *
 *   U1. Update ZK with RS = ORS + TRS, AR = TRS - ORS, RR = ORS - TRS.
 *   U2. Update memory with RS = ORS + TRS, AR = TRS - ORS and RR = ORS - TRS
 *   U3. If we are cancelling or replacing an existing reassignment, send StopReplica to all members
 *       of AR in the original reassignment if they are not in TRS from the new assignment
 *
 * To complete the reassignment, we need to bring the new replicas into sync, so depending on the state
 * of the ISR, we will execute one of the following steps.
 *
 * Phase A (when TRS != ISR): The reassignment is not yet complete
 *
 *   A1. Bump the leader epoch for the partition and send LeaderAndIsr updates to RS.
 *   A2. Start new replicas AR by moving replicas in AR to NewReplica state.
 *
 * Phase B (when TRS = ISR): The reassignment is complete
 *
 *   B1. Move all replicas in AR to OnlineReplica state.
 *   B2. Set RS = TRS, AR = [], RR = [] in memory.
 *   B3. Send a LeaderAndIsr request with RS = TRS. This will prevent the leader from adding any replica in TRS - ORS back in the isr.
 *       If the current leader is not in TRS or isn't alive, we move the leader to a new replica in TRS.
 *       We may send the LeaderAndIsr to more than the TRS replicas due to the
 *       way the partition state machine works (it reads replicas from ZK)
 *   B4. Move all replicas in RR to OfflineReplica state. As part of OfflineReplica state change, we shrink the
 *       isr to remove RR in ZooKeeper and send a LeaderAndIsr ONLY to the Leader to notify it of the shrunk isr.
 *       After that, we send a StopReplica (delete = false) to the replicas in RR.
 *   B5. Move all replicas in RR to NonExistentReplica state. This will send a StopReplica (delete = true) to
 *       the replicas in RR to physically delete the replicas on disk.
 *   B6. Update ZK with RS=TRS, AR=[], RR=[].
 *   B7. Remove the ISR reassign listener and maybe update the /admin/reassign_partitions path in ZK to remove this partition from it if present.
 *   B8. After electing leader, the replicas and isr information changes. So resend the update metadata request to every broker.
 *
 * In general, there are two goals we want to aim for:
 * 1. Every replica present in the replica set of a LeaderAndIsrRequest gets the request sent to it
 * 2. Replicas that are removed from a partition's assignment get StopReplica sent to them
 *
 * For example, if ORS = {1,2,3} and TRS = {4,5,6}, the values in the topic and leader/isr paths in ZK
 * may go through the following transitions.
 * RS                AR          RR          leader     isr
 * {1,2,3}           {}          {}          1          {1,2,3}           (initial state)
 * {4,5,6,1,2,3}     {4,5,6}     {1,2,3}     1          {1,2,3}           (step A2)
 * {4,5,6,1,2,3}     {4,5,6}     {1,2,3}     1          {1,2,3,4,5,6}     (phase B)
 * {4,5,6,1,2,3}     {4,5,6}     {1,2,3}     4          {1,2,3,4,5,6}     (step B3)
 * {4,5,6,1,2,3}     {4,5,6}     {1,2,3}     4          {4,5,6}           (step B4)
 * {4,5,6}           {}          {}          4          {4,5,6}           (step B6)
 *
 * Note that we have to update RS in ZK with TRS last since it's the only place where we store ORS persistently.
 * This way, if the controller crashes before that step, we can still recover.
 */
private def onPartitionReassignment(topicPartition: TopicPartition, reassignment: ReplicaAssignment): Unit = {
  // While a reassignment is in progress, deletion is not allowed
  topicDeletionManager.markTopicIneligibleForDeletion(Set(topicPartition.topic), reason = "topic reassignment in progress")

  updateCurrentReassignment(topicPartition, reassignment)

  val addingReplicas = reassignment.addingReplicas
  val removingReplicas = reassignment.removingReplicas

  if (!isReassignmentComplete(topicPartition, reassignment)) {
    // A1. Send LeaderAndIsr request to every replica in ORS + TRS (with the new RS, AR and RR).
    updateLeaderEpochAndSendRequest(topicPartition, reassignment)
    // A2. replicas in AR -> NewReplica
    startNewReplicasForReassignedPartition(topicPartition, addingReplicas)
  } else {
    // B1. replicas in AR -> OnlineReplica
    replicaStateMachine.handleStateChanges(addingReplicas.map(PartitionAndReplica(topicPartition, _)), OnlineReplica)
    // B2. Set RS = TRS, AR = [], RR = [] in memory.
    val completedReassignment = ReplicaAssignment(reassignment.targetReplicas)
    controllerContext.updatePartitionFullReplicaAssignment(topicPartition, completedReassignment)
    // B3. Send LeaderAndIsr request with a potential new leader (if current leader not in TRS) and
    //   a new RS (using TRS) and same isr to every broker in ORS + TRS or TRS
    moveReassignedPartitionLeaderIfRequired(topicPartition, completedReassignment)
    // B4. replicas in RR -> Offline (force those replicas out of isr)
    // B5. replicas in RR -> NonExistentReplica (force those replicas to be deleted)
    stopRemovedReplicasOfReassignedPartition(topicPartition, removingReplicas)
    // B6. Update ZK with RS = TRS, AR = [], RR = [].
    updateReplicaAssignmentForPartition(topicPartition, completedReassignment)
    // B7. Remove the ISR reassign listener and maybe update the /admin/reassign_partitions path in ZK to remove this partition from it.
    removePartitionFromReassigningPartitions(topicPartition, completedReassignment)
    // B8. After electing a leader in B3, the replicas and isr information changes, so resend the update metadata request to every broker
    sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq, Set(topicPartition))
    // signal delete topic thread if reassignment for some partitions belonging to topics being deleted just completed
    topicDeletionManager.resumeDeletionForTopics(Set(topicPartition.topic))
  }
}
```
