# 1.3 controller

Kafka 集群中会有一个 broker 作为集群 controller，负责一些管理和协调的工作：

* 监听 `/brokers/topics`，触发 topic 创建动作
* 监听 `/admin/delete_topics`，触发 topic 删除动作
* 监听 `/admin/reassign_partitions`，触发 topic 重分区操作
* 监听 `/admin/preferred_replica_election`，触发最优 leader 选举操作
* 监听 `/brokers/topics/<topic>`，触发 topic partition 扩容操作
* 监听 `/brokers/ids`，触发 broker 上线/下线 操作
* 当集群元信息变动时，controller 通过 `UpdateMetadataRequest` 请求广播信息到集群的所有 broker 上，通过 `LeaderAndIsrRequest`, `StopReplicaRequest` 发送请求到相关 broker
* broker 自行停止时，向 controller 发送 `ControlledShudownRequest` 请求，controller 负责收尾工作
* 监听 `/controller`，如果该节点消失，所有 broker 抢占 `/controller`，抢占成功则成为新的 controller


参考：
- https://cwiki.apache.org/confluence/display/kafka/kafka+controller+redesign
- https://jaceklaskowski.gitbooks.io/apache-kafka/content/kafka-controller-ControllerEventManager.html
- https://cwiki.apache.org/confluence/display/KAFKA/KIP-455%3A+Create+an+Administrative+API+for+Replica+Reassignment
