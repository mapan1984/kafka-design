# 1.4 ReplicaManager

每个 broker 进程有 1 个 ReplicaManager 实例，ReplicaManager 负责：

1. 定时检查副本是否落后，如果落后，需要通知 controller 将其移出 isr
2. 提供 `fetchMessages()` 方法，broker 处理来自 consumer 或者 follower 的 `FetchRequest` 时需要调用 replica manager 的 `fetchMessages()` 方法，`fetchMessages()` 会更新副本状态，必要时通知 controller 将副本加入 isr
3. 提供 `stopReplicas()` 方法，broker 处理来自 controller 的 `StopReplicaRequest` 时需要调用 replica manager 的 `stopReplicas()` 方法
4. 提供 `becomeLeaderOrFollower()` 方法，broker 处理来自 controller 的 `LeaderAndIsrRequest` 时需要调用 replica manager 的 `becomeLeaderOrFollower()` 方法
    1. 对于成为 leader 的本地 replica，调用 `makeLeaders()`
    2. 对于成为 follower 的本地 replica，调用 `makeFollowers()`，这里会创建并启动 fetcherThread
5. 提供 `maybeUpdateMetadataCache()` 方法，broker 处理来自 controller 的 `UpdateMetadataRequest` 时需要调用 replica manager 的 `maybeUpdateMetadataCache()` 方法
6. 提供 `appendRecords()` 方法，broker 处理来自 producer 的 `ProduceRequest` 时需要调用 replica manager 的 `appendRecords()` 方法
7. `ListOffsetRequest`
    1. handleListOffsetRequestV0:  提供 `legacyFetchOffsetsForTimestamp` 方法，broker 处理 `ListOffsetRequestV0` 请求时需要调用 replica manager 的 `legacyFetchOffsetsForTimestamp` 方法
    2. handleListOffsetRequestV1AndAbove:  提供 `fetchOffsetForTimestamp` 方法，broker 处理 `ListOffsetRequestV1` 请求时需要调用 replica manager 的 `fetchOffsetForTimestamp` 方法
8. 提供 `deleteRecords()` 方法，处理 `DeleteRecordsRequest` 请求
