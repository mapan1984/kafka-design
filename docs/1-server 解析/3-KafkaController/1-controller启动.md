# 1.3.1 controller 启动

controller 在 `kafkaServer.startup()` 中初始化并启动：

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

        /* start kafka controller */
        kafkaController = new KafkaController(config, zkClient, time, metrics, brokerInfo, brokerEpoch, tokenManager, brokerFeatures, featureCache, threadNamePrefix)
        kafkaController.startup()

        // ...
```

`KafkaController.startup()` 方法会向 zookeeper 注册 StateChangeHandler，并启动 eventManager

``` scala
/**
 * Invoked when the controller module of a Kafka server is started up. This does not assume that the current broker
 * is the controller. It merely registers the session expiration listener and starts the controller leader
 * elector
 */
def startup() = {
  zkClient.registerStateChangeHandler(new StateChangeHandler {
    override val name: String = StateChangeHandlers.ControllerHandler
    override def afterInitializingSession(): Unit = {
      eventManager.put(RegisterBrokerAndReelect)
    }
    override def beforeInitializingSession(): Unit = {
      val queuedEvent = eventManager.clearAndPut(Expire)

      // Block initialization of the new session until the expiration event is being handled,
      // which ensures that all pending events have been processed before creating the new session
      queuedEvent.awaitProcessing()
    }
  })
  eventManager.put(Startup)
  eventManager.start()
}
```

`eventManager` 是 `ControllerEventManager` 的实例，其维护了 `QueuedEvent`(对 `ControllerEvent` 的包装) 的队列，并启动线程：

1. 从 `queue` 中取事件 `QueuedEvent`
2. 通过 `processor` 对事件进行处理

这里的 `processor` 是 `KafkaController`，其实现了 `ControllerEventProcessor` 定义的方法 `process()` 方法

在启动 `eventManager` 之前，向 `eventManager` 增加了 `Startup` 事件，可以在 `KafkaController.process()` 方法中看到对所有事件的处理

``` scala
override def process(event: ControllerEvent): Unit = {
  try {
    event match {
      case event: MockEvent =>
        // Used only in test cases
        event.process()
      case ShutdownEventThread =>
        error("Received a ShutdownEventThread event. This type of event is supposed to be handle by ControllerEventThread")
      // ...
      case Startup =>
        processStartup()
    }
  } catch {
    case e: ControllerMovedException =>
      info(s"Controller moved to another broker when processing $event.", e)
      maybeResign()
    case e: Throwable =>
      error(s"Error processing event $event", e)
  } finally {
    updateMetrics()
  }
}
```

可以看到，对于 `Startup` 事件，这里实际通过 `processStartup()` 进行处理。

``` scala
private def processStartup(): Unit = {
  zkClient.registerZNodeChangeHandlerAndCheckExistence(controllerChangeHandler)
  elect()
}
```

这里首先向 zookeeper 注册 controllerChangeHandler 用来监听 `/controller`，负责 controller 自身的选举、管理操作：

``` scala
class ControllerChangeHandler(eventManager: ControllerEventManager) extends ZNodeChangeHandler {
  override val path: String = ControllerZNode.path

  override def handleCreation(): Unit = eventManager.put(ControllerChange)
  override def handleDeletion(): Unit = eventManager.put(Reelect)
  override def handleDataChange(): Unit = eventManager.put(ControllerChange)
}
```

- `/controller` zookeeper 节点创建时向 eventManager 的 queue 增加 `ControllerChange` 事件
- `/controller` zookeeper 节点删除时向 eventManager 的 queue 增加 `Reelect` 事件
- `/controller` zookeeper 节点内容改变时向 eventManager 的 queue 增加 `ControllerChange` 事件

然后，执行 `elect()`，尝试成为当前集群的 controller

``` scala
private def elect(): Unit = {
  activeControllerId = zkClient.getControllerId.getOrElse(-1)
  /*
   * We can get here during the initial startup and the handleDeleted ZK callback. Because of the potential race condition,
   * it's possible that the controller has already been elected when we get here. This check will prevent the following
   * createEphemeralPath method from getting into an infinite loop if this broker is already the controller.
   */
  if (activeControllerId != -1) {
    debug(s"Broker $activeControllerId has been elected as the controller, so stopping the election process.")
    return
  }

  try {
    val (epoch, epochZkVersion) = zkClient.registerControllerAndIncrementControllerEpoch(config.brokerId)
    controllerContext.epoch = epoch
    controllerContext.epochZkVersion = epochZkVersion
    activeControllerId = config.brokerId

    info(s"${config.brokerId} successfully elected as the controller. Epoch incremented to ${controllerContext.epoch} " +
      s"and epoch zk version is now ${controllerContext.epochZkVersion}")

    onControllerFailover()
  } catch {
    case e: ControllerMovedException =>
      maybeResign()

      if (activeControllerId != -1)
        debug(s"Broker $activeControllerId was elected as controller instead of broker ${config.brokerId}", e)
      else
        warn("A controller has been elected but just resigned, this will result in another round of election", e)
    case t: Throwable =>
      error(s"Error while electing or becoming controller on broker ${config.brokerId}. " +
        s"Trigger controller movement immediately", t)
      triggerControllerMove()
  }
}
```

1. 首先获取当前是否有 `/controller`，如果有，即 `activeControllerId != -1`，则返回，否则继续
2. 尝试以自身 brokerId 创建 `/controller`，如果创建成功，则当前 broker 为集群 controller，执行 `onControllerFailover()` 方法，如下：
    1. 初始化 controller context 缓存所有 topics，live brokers，leaders for all existing partitions
    2. 开启 controller channel manager
    3. 开启 replica state machine
    4. 开启 partition state machine

``` scala
/**
 * This callback is invoked by the zookeeper leader elector on electing the current broker as the new controller.
 * It does the following things on the become-controller state change -
 * 1. Initializes the controller's context object that holds cache objects for current topics, live brokers and
 *    leaders for all existing partitions.
 * 2. Starts the controller's channel manager
 * 3. Starts the replica state machine
 * 4. Starts the partition state machine
 * If it encounters any unexpected exception/error while becoming controller, it resigns as the current controller.
 * This ensures another controller election will be triggered and there will always be an actively serving controller
 */
private def onControllerFailover(): Unit = {
  maybeSetupFeatureVersioning()

  info("Registering handlers")

  // before reading source of truth from zookeeper, register the listeners to get broker/topic callbacks
  val childChangeHandlers = Seq(brokerChangeHandler, topicChangeHandler, topicDeletionHandler, logDirEventNotificationHandler,
    isrChangeNotificationHandler)
  childChangeHandlers.foreach(zkClient.registerZNodeChildChangeHandler)

  val nodeChangeHandlers = Seq(preferredReplicaElectionHandler, partitionReassignmentHandler)
  nodeChangeHandlers.foreach(zkClient.registerZNodeChangeHandlerAndCheckExistence)

  info("Deleting log dir event notifications")
  zkClient.deleteLogDirEventNotifications(controllerContext.epochZkVersion)
  info("Deleting isr change notifications")
  zkClient.deleteIsrChangeNotifications(controllerContext.epochZkVersion)
  info("Initializing controller context")
  initializeControllerContext()
  info("Fetching topic deletions in progress")
  val (topicsToBeDeleted, topicsIneligibleForDeletion) = fetchTopicDeletionsInProgress()
  info("Initializing topic deletion manager")
  topicDeletionManager.init(topicsToBeDeleted, topicsIneligibleForDeletion)

  // We need to send UpdateMetadataRequest after the controller context is initialized and before the state machines
  // are started. The is because brokers need to receive the list of live brokers from UpdateMetadataRequest before
  // they can process the LeaderAndIsrRequests that are generated by replicaStateMachine.startup() and
  // partitionStateMachine.startup().
  info("Sending update metadata request")
  sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq, Set.empty)

  replicaStateMachine.startup()
  partitionStateMachine.startup()

  info(s"Ready to serve as the new controller with epoch $epoch")

  initializePartitionReassignments()
  topicDeletionManager.tryTopicDeletion()
  val pendingPreferredReplicaElections = fetchPendingPreferredReplicaElections()
  onReplicaElection(pendingPreferredReplicaElections, ElectionType.PREFERRED, ZkTriggered)
  info("Starting the controller scheduler")
  kafkaScheduler.startup()
  if (config.autoLeaderRebalanceEnable) {
    scheduleAutoLeaderRebalanceTask(delay = 5, unit = TimeUnit.SECONDS)
  }

  if (config.tokenAuthEnabled) {
    info("starting the token expiry check scheduler")
    tokenCleanScheduler.startup()
    tokenCleanScheduler.schedule(name = "delete-expired-tokens",
      fun = () => tokenManager.expireTokens(),
      period = config.delegationTokenExpiryCheckIntervalMs,
      unit = TimeUnit.MILLISECONDS)
  }
}
```

1. 注册以下 handler，监听 zookeeper，做对应处理：
    1. `brokerChangeHandler`，监听 `/brokers/ids`，如果 child 有变化，向 eventManager 增加 `BrokerChange` 事件
    2. `topicChangeHandler`，监听 `/brokers/topics`，如果 child 有变化，向 eventManager 增加 `TopicChange` 事件
    3. `topicDeletionHandler`，监听 `/admin/delete_topics`，如果 child 有变化，向 eventManager 增加 `TopicDeletion` 事件
    4. `logDirEventNotificationHandler`，监听 `/log_dir_event_notification`，如果 child 有变化，向 eventManager 增加 `LogDirEventNotification` 事件
    5. `isrChangeNotificationHandler`，监听 `/isr_change_notification`，如果 child 有变化，向 eventManager 增加 `IsrChangeNotification` 事件
    6. `preferredReplicaElectionHandler`，监听 `/admin/preferred_replica_election`，如果新建，向 eventManager 增加 `ReplicaLeaderElection` 事件
    7. `partitionReassignmentHandler`，监听 `/admin/reassign_partitions`，如果新建，向 eventManager 增加 `ZkPartitionReassignment` 事件
2. 初始化 controllerContext
3. 获取 `/admin/delete_topics/` 节点，查看是否有待删除的 topic，初始 TopicDeletionManager
4. 向所有 broker 发送 UpdateMetadataRequest
5. 开启 replicaStateMachine
6. 开启 partitionStateMachine
7. 尝试进行未完成的 partitionReassignments
8. 尝试进行未完成的 preferredReplicaElections
9. 启动 kafkaScheduler
10. 如果配置了 auto leader rebalance，则周期进行 leader rebalance
11. 如果开启 token auth，则周期删除过期 token

至此，controller 已经完成启动
