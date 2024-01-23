# 5 release note

感兴趣的 KIP

## 重分区 API KIP-455

增加了重分区区的 API，相比基于 zookeeper 的方式，这种重分区可以终止，可以在上一个还在执行时增加任务

https://cwiki.apache.org/confluence/display/KAFKA/KIP-455%3A+Create+an+Administrative+API+for+Replica+Reassignment

## kafka consumer 支持从 replica 拉数据 KIP-392

https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica

## controller 支持 AlterIsrRequest KIP-497

可以不再依赖 zookeeper `/isr_change_notification` 传递 isr 变化信息

https://cwiki.apache.org/confluence/display/KAFKA/KIP-497%3A+Add+inter-broker+API+to+alter+ISR

## 消费组在单独的线程发送心跳 KIP-62

https://cwiki.apache.org/confluence/display/KAFKA/KIP-62%3A+Allow+consumer+to+send+heartbeats+from+a+background+thread

## 分区数限制 KIP-578

https://cwiki.apache.org/confluence/display/KAFKA/KIP-578%3A+Add+configuration+to+limit+number+of+partitions

当前推荐值是每个 broker 不超过 4000 个分区，每个集群不超过 200000 个分区，但是没有强制限制。

可以通过配置 `create.topic.policy.class.name` 为特定的 topic 创建策略实现，在分区数过多时拒绝 topic 创建请求。

后续计划增加参数：

* `max.broker.partitions`：单个 broker 最多 partition
* `max.partitions`: 集群最多 partition

## 去除对 ZooKeeper 集群的依赖 KIP-500

https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum

## 支持在 log_dirs 之间移动副本目录 KIP-113

https://cwiki.apache.org/confluence/display/KAFKA/KIP-113%3A+Support+replicas+movement+between+log+directories

* 增加 `AlterReplicaDirRequest` 请求
* 增加 `DescribeLogDirsRequest` 请求

## 多磁盘下单个磁盘不可用仍然可提供服务 KIP-112

https://cwiki.apache.org/confluence/display/KAFKA/KIP-112%3A+Handle+disk+failure+for+JBOD

* 增加 `/log_dir_event_notification` znode 节点，向 controller 传递 `LogDirFailure` 事件

## 支持机架副本分布 KIP-36

https://cwiki.apache.org/confluence/display/KAFKA/KIP-36+Rack+aware+replica+assignment
