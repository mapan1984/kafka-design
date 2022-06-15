# 1.6.1 LogManager 启动

在 `kafkaServer.startup()` 中：

``` scala
/* start log manager */
logManager = LogManager(config, initialOfflineDirs, zkClient, brokerState, kafkaScheduler, time, brokerTopicStats, logDirFailureChannel)
logManager.startup()
```

实际调用 `LogManager.apply()` 创建 `LogManager` 对象：

``` scala
object LogManager {

  val RecoveryPointCheckpointFile = "recovery-point-offset-checkpoint"
  val LogStartOffsetCheckpointFile = "log-start-offset-checkpoint"
  val ProducerIdExpirationCheckIntervalMs = 10 * 60 * 1000

  def apply(config: KafkaConfig,
            initialOfflineDirs: Seq[String],
            zkClient: KafkaZkClient,
            brokerState: BrokerState,
            kafkaScheduler: KafkaScheduler,
            time: Time,
            brokerTopicStats: BrokerTopicStats,
            logDirFailureChannel: LogDirFailureChannel): LogManager = {
    val defaultProps = KafkaServer.copyKafkaConfigToLog(config)
    val defaultLogConfig = LogConfig(defaultProps)

    // read the log configurations from zookeeper
    val (topicConfigs, failed) = zkClient.getLogConfigs(zkClient.getAllTopicsInCluster, defaultProps)
    if (!failed.isEmpty) throw failed.head._2

    val cleanerConfig = LogCleaner.cleanerConfig(config)

    new LogManager(logDirs = config.logDirs.map(new File(_).getAbsoluteFile),
      initialOfflineDirs = initialOfflineDirs.map(new File(_).getAbsoluteFile),
      topicConfigs = topicConfigs,
      initialDefaultConfig = defaultLogConfig,
      cleanerConfig = cleanerConfig,
      recoveryThreadsPerDataDir = config.numRecoveryThreadsPerDataDir,
      flushCheckMs = config.logFlushSchedulerIntervalMs,
      flushRecoveryOffsetCheckpointMs = config.logFlushOffsetCheckpointIntervalMs,
      flushStartOffsetCheckpointMs = config.logFlushStartOffsetCheckpointIntervalMs,
      retentionCheckMs = config.logCleanupIntervalMs,
      maxPidExpirationMs = config.transactionIdExpirationMs,
      scheduler = kafkaScheduler,
      brokerState = brokerState,
      brokerTopicStats = brokerTopicStats,
      logDirFailureChannel = logDirFailureChannel,
      time = time)
  }
}
```

`LogManager` 负责：
* creation
* retrieval
* cleaning

所有 read 和 write 操作都会由单独的 `Log` 对象负责

`LogManager` 维护一个或多个目录，新的 log 会在 log 最少的目录中创建，没有尝试移动分区，考虑 size 或者 I/O rate 均衡。

一个后台线程通过周期回收 log segment 处理 log retention

`LogManager.startup()` 启动实际的后台线程：

``` scala
/**
 *  Start the background threads to flush logs and do log cleanup
 */
def startup() {
  /* Schedule the cleanup task to delete old logs */
  if (scheduler != null) {
    info("Starting log cleanup with a period of %d ms.".format(retentionCheckMs))
    scheduler.schedule("kafka-log-retention",
                       cleanupLogs _,
                       delay = InitialTaskDelayMs,
                       period = retentionCheckMs,
                       TimeUnit.MILLISECONDS)
    info("Starting log flusher with a default period of %d ms.".format(flushCheckMs))
    scheduler.schedule("kafka-log-flusher",
                       flushDirtyLogs _,
                       delay = InitialTaskDelayMs,
                       period = flushCheckMs,
                       TimeUnit.MILLISECONDS)
    scheduler.schedule("kafka-recovery-point-checkpoint",
                       checkpointLogRecoveryOffsets _,
                       delay = InitialTaskDelayMs,
                       period = flushRecoveryOffsetCheckpointMs,
                       TimeUnit.MILLISECONDS)
    scheduler.schedule("kafka-log-start-offset-checkpoint",
                       checkpointLogStartOffsets _,
                       delay = InitialTaskDelayMs,
                       period = flushStartOffsetCheckpointMs,
                       TimeUnit.MILLISECONDS)
    scheduler.schedule("kafka-delete-logs", // will be rescheduled after each delete logs with a dynamic period
                       deleteLogs _,
                       delay = InitialTaskDelayMs,
                       unit = TimeUnit.MILLISECONDS)
  }
  // 如果设置为 true，自动清理 compaction 类型的 topic
  if (cleanerConfig.enableCleaner)
    cleaner.startup()
}
```

## loadLogs

`LogManager` 实例化时会通过 `loadLogs()` 方法在配置的 `log.dirs` 目录中加载所有的 log


num.recovery.threads.per.data.dir

