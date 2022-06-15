# 1.3.2.3 PartitionStateMachine

partition 所有状态：

1. `NonExistentPartition`: 这个状态表示 parition 没有被创建，或者曾创建过但被删除了。有效的前置状态是 `OfflinePartition`
2. `NewPartition`: 创建之后，partition 处于 `NewPartition` 状态。在此状态下，partition 应该有 assigned replicas，但是没有 leader 和 isr。有效的前置状态是 `NonExistentPartition`
3. `OnlinePartition`: 当 partition 的 leader 选举出来，其处于 `OnlinePartition` 状态。有效的前置状态是 `NewPartition`, `OfflinePartition`
4. `OfflinePartition`: 如果经过成功的 leader 选举之后，leader 因为一些原因下线了，partition 此时处于 `OfflinePartition` 状态。有效的前置状态是 `NewPartition`, `OnlinePartition`

``` scala
sealed trait PartitionState {
  def state: Byte
  def validPreviousStates: Set[PartitionState]
}

case object NewPartition extends PartitionState {
  val state: Byte = 0
  val validPreviousStates: Set[PartitionState] = Set(NonExistentPartition)
}

case object OnlinePartition extends PartitionState {
  val state: Byte = 1
  val validPreviousStates: Set[PartitionState] = Set(NewPartition, OnlinePartition, OfflinePartition)
}

case object OfflinePartition extends PartitionState {
  val state: Byte = 2
  val validPreviousStates: Set[PartitionState] = Set(NewPartition, OnlinePartition, OfflinePartition)
}

case object NonExistentPartition extends PartitionState {
  val state: Byte = 3
  val validPreviousStates: Set[PartitionState] = Set(OfflinePartition)
}
```
