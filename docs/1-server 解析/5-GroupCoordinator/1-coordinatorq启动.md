# 1.5.1 coordinator 启动

coordinator 在 `kafkaServer.startup()` 中启动：

``` scala
        /* start group coordinator */
        // Hardcode Time.SYSTEM for now as some Streams tests fail otherwise, it would be good to fix the underlying issue
        groupCoordinator = GroupCoordinator(config, zkClient, replicaManager, Time.SYSTEM, metrics)
        groupCoordinator.startup()
```

groupCoordinator 的 `startup()` 方法如下：

``` scala
  /**
   * Startup logic executed at the same time when the server starts up.
   */
  def startup(enableMetadataExpiration: Boolean = true): Unit = {
    info("Starting up.")
    groupManager.startup(enableMetadataExpiration)
    isActive.set(true)
    info("Startup complete.")
  }
```

groupManager 的 `startup()` 方法如下：

``` scala
  def startup(enableMetadataExpiration: Boolean): Unit = {
    scheduler.startup()
    if (enableMetadataExpiration) {
      scheduler.schedule(name = "delete-expired-group-metadata",
        fun = () => cleanupGroupMetadata(),
        period = config.offsetsRetentionCheckIntervalMs,
        unit = TimeUnit.MILLISECONDS)
    }
  }
```

实际上启动了一个后台线程，并且定时任务，每隔 `offsets.retention.check.interval.ms` 毫秒清理超过 `offsets.retention.minutes` 的 offset cache
