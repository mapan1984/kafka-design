# 1.4.1 replica manager 启动

## 创建

replica manager 在 `kafkaServer.startup()` 中初始化并启动：

``` scala
  /**
   * Start up API for bringing up a single instance of the Kafka server.
   * Instantiates the LogManager, the SocketServer and the request handlers - KafkaRequestHandlers
   */
  def startup(): Unit = {
    try {
      info("starting")

      if (isShuttingDown.get)
        throw new IllegalStateException("Kafka server is still shutting down, cannot re-start!")

      if (startupComplete.get)
        return

      val canStartup = isStartingUp.compareAndSet(false, true)
      if (canStartup) {

        // ...

        /* start replica manager */
        brokerToControllerChannelManager = new BrokerToControllerChannelManagerImpl(metadataCache, time, metrics, config, threadNamePrefix)
        replicaManager = createReplicaManager(isShuttingDown)
        replicaManager.startup()
        brokerToControllerChannelManager.start()

        // ...
```

``` scala
  protected def createReplicaManager(isShuttingDown: AtomicBoolean): ReplicaManager = {
    val alterIsrManager = new AlterIsrManagerImpl(brokerToControllerChannelManager, kafkaScheduler,
      time, config.brokerId, () => kafkaController.brokerEpoch)
    new ReplicaManager(config, metrics, time, zkClient, kafkaScheduler, logManager, isShuttingDown, quotaManagers,
      brokerTopicStats, metadataCache, logDirFailureChannel, alterIsrManager)
  }
```

## startup

`ReplicaManager.startup()` 方法会启动以下定时任务：

1. isr-expiration：定时检查 isr 列表中是否有 replica 需要被从 isr 列表中移除，这里会对 isr 变化的 topicPartition 进行记录，分为 2 种情况：
    1. 如果 api version `>=` KAFKA_2_7_IV2，记录在 alterIsrManager 的 unsentIsrUpdates 中
    2. 否则记录在自身的 isrChangeSet 中
2. isr 的变化需要通知到 controller，上一步的定时任务只是记录，这里会根据集群支持的 API version 分别通过 AlterIsrRequest 请求或者 zookeeper 向 controller 进行通知:
    1. send-alter-isr: 如果 api version `>=` KAFKA_2_7_IV2，启动 alterIsrManager，alterIsrManager 同样启动定时任务，定期向 controller 发送未完成的 AlterIsrRequest 请求
    2. isr-change-propagation: 否则定时检查 isr 是否需要变动，创建 `/isr_change_notification/isr_change_<>` zookeeper 节点，触发 controller 动作
3. shutdown-idle-replica-alter-log-dirs-thread

``` scala
  def startup(): Unit = {
    // start ISR expiration thread
    // A follower can lag behind leader for up to config.replicaLagTimeMaxMs x 1.5 before it is removed from ISR
    scheduler.schedule("isr-expiration", maybeShrinkIsr _, period = config.replicaLagTimeMaxMs / 2, unit = TimeUnit.MILLISECONDS)
    // If using AlterIsr, we don't need the znode ISR propagation
    if (!config.interBrokerProtocolVersion.isAlterIsrSupported) {
      scheduler.schedule("isr-change-propagation", maybePropagateIsrChanges _,
        period = isrChangeNotificationConfig.checkIntervalMs, unit = TimeUnit.MILLISECONDS)
    } else {
      alterIsrManager.start()
    }
    scheduler.schedule("shutdown-idle-replica-alter-log-dirs-thread", shutdownIdleReplicaAlterLogDirsThread _, period = 10000L, unit = TimeUnit.MILLISECONDS)

    // If inter-broker protocol (IBP) < 1.0, the controller will send LeaderAndIsrRequest V0 which does not include isNew field.
    // In this case, the broker receiving the request cannot determine whether it is safe to create a partition if a log directory has failed.
    // Thus, we choose to halt the broker on any log diretory failure if IBP < 1.0
    val haltBrokerOnFailure = config.interBrokerProtocolVersion < KAFKA_1_0_IV0
    logDirFailureHandler = new LogDirFailureHandler("LogDirFailureHandler", haltBrokerOnFailure)
    logDirFailureHandler.start()
  }
```

