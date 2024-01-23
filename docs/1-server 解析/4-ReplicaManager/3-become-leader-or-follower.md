# 1.4.3 LeaderAndIsrRequest

`becomeLeaderOrFollower` 处理 LeaderAndIsrRequest 请求。

ReplicaManager 维护的 `allPartitions` 中的 topic partition 记录是在这里添加的。

* broker 启动后，controller 会向 broker 发送 `LeaderAndIsrRequest` 请求
* isr 变化时，

## leader epoch

partition leader 每切换一次，epoch 增加 1。

leader-epoch-checkpoint 记录每次 leader 切换后的 epoch 以及该次切换后 leader 写入第一条消息的 offset（该 partition 的 log end offset）。

