# 1.3.2.1 ControllerContext

`ControllerContext` 包含 Controller 需要维护的信息：

``` scala
// 下线分区数
var offlinePartitionCount = 0

var preferredReplicaImbalanceCount = 0

val shuttingDownBrokerIds = mutable.Set.empty[Int]

private val liveBrokers = mutable.Set.empty[Broker]

private val liveBrokerEpochs = mutable.Map.empty[Int, Long]

var epoch: Int = KafkaController.InitialControllerEpoch

var epochZkVersion: Int = KafkaController.InitialControllerEpochZkVersion

val allTopics = mutable.Set.empty[String]

// 记录 topic 对应的 partition 对应的 replicaAssignments
val partitionAssignments = mutable.Map.empty[String, mutable.Map[Int, ReplicaAssignment]]

// 记录 topic partition 与其对应的 leader isr
private val partitionLeadershipInfo = mutable.Map.empty[TopicPartition, LeaderIsrAndControllerEpoch]

val partitionsBeingReassigned = mutable.Set.empty[TopicPartition]

// 记录 topicPartition 对应的 状态
val partitionStates = mutable.Map.empty[TopicPartition, PartitionState]

// 记录 topicPartition 对应的 replica 状态
val replicaStates = mutable.Map.empty[PartitionAndReplica, ReplicaState]

val replicasOnOfflineDirs = mutable.Map.empty[Int, Set[TopicPartition]]

val topicsToBeDeleted = mutable.Set.empty[String]
```

