# ReplicationQuotaManager

实现 `ReplicaQuota` trait

``` scala
trait ReplicaQuota {
  def record(value: Long): Unit
  def isThrottled(topicPartition: TopicPartition): Boolean
  def isQuotaExceeded: Boolean
}
```

``` scala
// key 为 topic，value 为 partition 列表，通过 markThrottled() 方法添加记录
private val throttledPartitions = new ConcurrentHashMap[String, Seq[Int]]()

private var quota: Quota = null

private val sensorAccess = new SensorAccess(lock, metrics)

private val rateMetricName = metrics.metricName("byte-rate", replicationType.toString,
  s"Tracking byte-rate for ${replicationType}")
```

