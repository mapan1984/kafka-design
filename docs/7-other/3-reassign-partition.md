# kafka reassign partition

## zk 管理方式

直接将分区分配方案写入到 ZooKeeper 的 `/admin/reassign_partitions` 节点，由 Controller 监听到后执行。

* 写入后，无法取消任务
* 写入后，无法增加新的分区分配操作

* generateAssignment
* executeAssignment
* verifyAssignment

## 增加 admin api

分区分配任务运行中，可以支持增加新的分配操作，支持取消（恢复）分区分配任务。

信息存储在 ZooKeeper 的 `/brokers/topics/[topic]` 节点中。

* generateAssignment
* executeAssignment
* verifyAssignment
* cancelAssignment
* listReassignments

增加 2 个新的 API：

* `alterPartitionAssignments`
    * 改变，可以新增，也可以减少
    * 提供空的分配计划将会取消分区分配
* `listPartitionReassignments`

## 参考

* KIP-455: 增加管理 API 用来分区重分配：https://cwiki.apache.org/confluence/display/KAFKA/KIP-455%3A+Create+an+Administrative+API+for+Replica+Reassignment
