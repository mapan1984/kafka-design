# 1.6.1 LogManager 启动

## 创建

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

## 启动

### loadLogs

`LogManager` 实例化时会通过 `loadLogs()` 方法遍历 `log.dirs` 配置的每个目录，对每个目前启动 `num.recovery.threads.per.data.dir` 个线程，加载目录下的数据文件。

`log.dirs` 配置目录下的每个 logDir 目录代表一个 topic partition，使用 `loadLog()` 加载。

`loadLog()` 会创建一个 `Log` 对象实例，并将 topic partition 与对应的 `Log` 实例保存到 `currentLogs` 中。

#### Log 对象

`Log` 对象在初始化时，会通过 `loadSegments()` 方法加载 topic partition 目录下所有 segments。

### startup

`LogManager.startup()` 启动的后台线程：

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

