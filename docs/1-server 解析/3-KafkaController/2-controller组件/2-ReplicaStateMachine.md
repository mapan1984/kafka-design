# 1.3.2.2 ReplicaStateMachine

ReplicaStateMachine 提供接口，负责处理 replica 状态变更。replica 状态缓存在 ControllerContext partitionAssignments 中，状态变更过程需要与 zookeeper 以及 broker 交互。

replica 所有状态：

1. `NewReplica`: controller 在 partition reassignment 期间可以创建新的 replicas。在此状态下，replica 只能成为 follower。有效的前置状态是 `NonExistentReplica`
2. `OnlineReplica`: 当 replica 成为其 partition 的 assigned replicas 的一部分，replica 处于这个状态。在此状态下，replica 可以成为 leader 或者 follower。有效的前置状态有 `NewReplica`, `OnlineReplica`, `OfflineReplica`
3. `OfflineReplica`: 如果 replica 所处 broker 下线，replica 处于这个状态。有效的前置状态是 `NewReplica`, `OnlineReplica`
4. `ReplicaDeletionStarted`: 如果 replica 开始删除，replica 会处于这个状态。有效的前置状态是 `OfflineReplica`
5. `ReplicaDeletionSuccessful`: 如果 delete replica 请求的响应没有错误，replica 会处于这个状态。有效的前置状态是 `ReplicaDeletionStarted`
6. `ReplicaDeletionIneligible`: 如果 replica 删除失败，replica 会处于这个状态。有效的前置状态是 `ReplicaDeletionStarted`, `OfflineReplica`
7. `NonExistentReplica`: 如果 replica 删除成功，replica 会处于这个状态。有效的前置状态是 `ReplicaDeletionSuccessful`


``` scala
sealed trait ReplicaState {
  def state: Byte
  def validPreviousStates: Set[ReplicaState]
}

case object NewReplica extends ReplicaState {
  val state: Byte = 1
  val validPreviousStates: Set[ReplicaState] = Set(NonExistentReplica)
}

case object OnlineReplica extends ReplicaState {
  val state: Byte = 2
  val validPreviousStates: Set[ReplicaState] = Set(NewReplica, OnlineReplica, OfflineReplica, ReplicaDeletionIneligible)
}

case object OfflineReplica extends ReplicaState {
  val state: Byte = 3
  val validPreviousStates: Set[ReplicaState] = Set(NewReplica, OnlineReplica, OfflineReplica, ReplicaDeletionIneligible)
}

case object ReplicaDeletionStarted extends ReplicaState {
  val state: Byte = 4
  val validPreviousStates: Set[ReplicaState] = Set(OfflineReplica)
}

case object ReplicaDeletionSuccessful extends ReplicaState {
  val state: Byte = 5
  val validPreviousStates: Set[ReplicaState] = Set(ReplicaDeletionStarted)
}

case object ReplicaDeletionIneligible extends ReplicaState {
  val state: Byte = 6
  val validPreviousStates: Set[ReplicaState] = Set(OfflineReplica, ReplicaDeletionStarted)
}

case object NonExistentReplica extends ReplicaState {
  val state: Byte = 7
  val validPreviousStates: Set[ReplicaState] = Set(ReplicaDeletionSuccessful)
}
```

