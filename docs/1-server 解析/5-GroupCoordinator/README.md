# 1.5 GroupCoordinator

kafka 集群每个 broker 都会启动一个 GroupCoordinator，负责 consumer group member 与订阅 topic partition 的分配与 offset 管理。

每个 GroupCoordinator 都会负责一组 group，由 group id 决定 group 该由那个 GroupCoordinator 负责，具体分配方法可以查看 `handleFindCoordinatorRequest()` 方法。

实际是通过 group id 找到 `__consumer_offsets` 这个 topic 对应的 partition，以 partition leader 所在节点作为该 group id 的 GroupCoordinator。

``` scala
def partitionFor(groupId: String): Int = Utils.abs(groupId.hashCode) % groupMetadataTopicPartitionCount

public static int abs(int n) {
    return (n == Integer.MIN_VALUE) ? 0 : Math.abs(n);
}
```

`groupMetadataTopicPartitionCount` 是 `offsets.topic.num.partitions` 参数的值，是 `__consumer_offsets` partition 的数量，默认是 50
