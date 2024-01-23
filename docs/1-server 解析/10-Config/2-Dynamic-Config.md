# 动态配置

配置存储在 zk 中，分 4 种类型：

- topics
- clients
- users
- brokers

## 配置定义

### DynamicBrokerConfig

属性：

``` scala
  // 记录当前 broker 动态配置
  private val dynamicBrokerConfigs = mutable.Map[String, String]()

  // 记录 <default> 即集群级别动态配置
  private val dynamicDefaultConfigs = mutable.Map[String, String]()

  // 记录所有可以重新配置的对象
  private val reconfigurables = mutable.Buffer[Reconfigurable]()

  // 记录所有可以重新配置的 broker 对象
  private val brokerReconfigurables = mutable.Buffer[BrokerReconfigurable]()

  // 当前的 kafkaConfig
  private var currentConfig = kafkaConfig
```

记录所有可以重新配置的对象

``` scala
  /**
   * Add reconfigurables to be notified when a dynamic broker config is updated.
   *
   * `Reconfigurable` is the public API used by configurable plugins like metrics reporter
   * and quota callbacks. These are reconfigured before `KafkaConfig` is updated so that
   * the update can be aborted if `reconfigure()` fails with an exception.
   *
   * `BrokerReconfigurable` is used for internal reconfigurable classes. These are
   * reconfigured after `KafkaConfig` is updated so that they can access `KafkaConfig`
   * directly. They are provided both old and new configs.
   */
  def addReconfigurables(kafkaServer: KafkaServer): Unit = {
    kafkaServer.authorizer match {
      case Some(authz: Reconfigurable) => addReconfigurable(authz)
      case _ =>
    }
    addReconfigurable(kafkaServer.kafkaYammerMetrics)
    addReconfigurable(new DynamicMetricsReporters(kafkaConfig.brokerId, kafkaServer))
    addReconfigurable(new DynamicClientQuotaCallback(kafkaConfig.brokerId, kafkaServer))

    addBrokerReconfigurable(new DynamicThreadPool(kafkaServer))
    if (kafkaServer.logManager.cleaner != null)
      addBrokerReconfigurable(kafkaServer.logManager.cleaner)
    addBrokerReconfigurable(new DynamicLogConfig(kafkaServer.logManager, kafkaServer))
    addBrokerReconfigurable(new DynamicListenerConfig(kafkaServer))
    addBrokerReconfigurable(kafkaServer.socketServer)
  }
```

## 配置初始化

在 `KafkaServer.startup()` 方法中调用 `DynamicBrokerConfig.initialize()` 方法初始化

``` scala
        // NOTE: 这里 initialize，但是 reconfigurables 是空的，所以这里只会更新 dynamicConfig 的 currentConfig，
        // 但是实际的 reconfigurable 对象并不会重新配置
        // 但是会更新 kafkaConfig 的 currentConfig
        // initialize dynamic broker configs from ZooKeeper. Any updates made after this will be
        // applied after DynamicConfigManager starts.
        config.dynamicConfig.initialize(zkClient)
```

1. 这里执行 `DynamicBrokerConfig.initialize()` 方法，但是 reconfigurables 是空的，所以这里只会更新 dynamicConfig 的 currentConfig，但是实际的 reconfigurable 对象并不会重新配置
2. 因为更新了自身的 `currentConfig`，动态的配置值获取方式可以从 `currentConfig` 中动态获取

``` scala
  private[server] def initialize(zkClient: KafkaZkClient): Unit = {
    currentConfig = new KafkaConfig(kafkaConfig.props, false, None)
    val adminZkClient = new AdminZkClient(zkClient)
    updateDefaultConfig(adminZkClient.fetchEntityConfig(ConfigType.Broker, ConfigEntityName.Default))
    val props = adminZkClient.fetchEntityConfig(ConfigType.Broker, kafkaConfig.brokerId.toString)
    val brokerConfig = maybeReEncodePasswords(props, adminZkClient)
    updateBrokerConfig(kafkaConfig.brokerId, brokerConfig)
  }


  private[server] def updateBrokerConfig(brokerId: Int, persistentProps: Properties): Unit = CoreUtils.inWriteLock(lock) {
    try {
      val props = fromPersistentProps(persistentProps, perBrokerConfig = true)
      dynamicBrokerConfigs.clear()
      dynamicBrokerConfigs ++= props.asScala
      updateCurrentConfig()
    } catch {
      case e: Exception => error(s"Per-broker configs of $brokerId could not be applied: $persistentProps", e)
    }
  }

  private[server] def updateDefaultConfig(persistentProps: Properties): Unit = CoreUtils.inWriteLock(lock) {
    try {
      val props = fromPersistentProps(persistentProps, perBrokerConfig = false)
      dynamicDefaultConfigs.clear()
      dynamicDefaultConfigs ++= props.asScala
      updateCurrentConfig()
    } catch {
      case e: Exception => error(s"Cluster default configs could not be applied: $persistentProps", e)
    }
  }

  private def updateCurrentConfig(): Unit = {
    val newProps = mutable.Map[String, String]()
    newProps ++= staticBrokerConfigs
    overrideProps(newProps, dynamicDefaultConfigs)
    overrideProps(newProps, dynamicBrokerConfigs)
    val oldConfig = currentConfig
    val (newConfig, brokerReconfigurablesToUpdate) = processReconfiguration(newProps, validateOnly = false)

    if (newConfig ne currentConfig) {
      currentConfig = newConfig
      kafkaConfig.updateCurrentConfig(newConfig)

      // Process BrokerReconfigurable updates after current config is updated
      brokerReconfigurablesToUpdate.foreach(_.reconfigure(oldConfig, newConfig))
    }
  }
```

`KafkaConfig` 的 `currentConfig` 更新后，自身的 `originals`, `values` 等值都从 `currentConfig` 中获取

``` scala
  private[server] def updateCurrentConfig(newConfig: KafkaConfig): Unit = {
    this.currentConfig = newConfig
  }

  override def originals: util.Map[String, AnyRef] =
    if (this eq currentConfig) super.originals else currentConfig.originals
  override def values: util.Map[String, _] =
    if (this eq currentConfig) super.values else currentConfig.values
  override def originalsStrings: util.Map[String, String] =
    if (this eq currentConfig) super.originalsStrings else currentConfig.originalsStrings
  override def originalsWithPrefix(prefix: String): util.Map[String, AnyRef] =
    if (this eq currentConfig) super.originalsWithPrefix(prefix) else currentConfig.originalsWithPrefix(prefix)
  override def valuesWithPrefixOverride(prefix: String): util.Map[String, AnyRef] =
    if (this eq currentConfig) super.valuesWithPrefixOverride(prefix) else currentConfig.valuesWithPrefixOverride(prefix)
  override def get(key: String): AnyRef =
    if (this eq currentConfig) super.get(key) else currentConfig.get(key)
```

静态配置用 `val` 定义为常量，`KafkaConfig` 实例化时初始一次。动态配置用 `def` 定义为方法，每次获取都执行一次，返回的值可以随着 `currentConfig` 的更新而更新

``` scala
  // 静态配置
  val hostName = getString(KafkaConfig.HostNameProp)
  val port = getInt(KafkaConfig.PortProp)
  val advertisedHostName = Option(getString(KafkaConfig.AdvertisedHostNameProp)).getOrElse(hostName)
  val advertisedPort: java.lang.Integer = Option(getInt(KafkaConfig.AdvertisedPortProp)).getOrElse(port)

  // NOTE: 动态配置
  def maxConnections = getInt(KafkaConfig.MaxConnectionsProp)
  def maxConnectionCreationRate = getInt(KafkaConfig.MaxConnectionCreationRateProp)
```

## 配置更新

在 `KafkaServer.startup()` 方法中

``` scala
        // NOTE: 注意这里，addReconfigurables 在 initialize 之后，所以之前 initialize 时不会对 reconfigurable 应用动态配置，
        // 但是实际的 reconfigurable 对象并不会重新配置
        // 但是会更新 kafkaConfig 的 currentConfig

        /* Add all reconfigurables for config change notification before starting config handlers */
        config.dynamicConfig.addReconfigurables(this)

        /* start dynamic config manager */
        dynamicConfigHandlers = Map[String, ConfigHandler](ConfigType.Topic -> new TopicConfigHandler(logManager, config, quotaManagers, kafkaController),
                                                           ConfigType.Client -> new ClientIdConfigHandler(quotaManagers),
                                                           ConfigType.User -> new UserConfigHandler(quotaManagers, credentialProvider),
                                                           ConfigType.Broker -> new BrokerConfigHandler(config, quotaManagers))

        // Create the config manager. start listening to notifications
        dynamicConfigManager = new DynamicConfigManager(zkClient, dynamicConfigHandlers)
        dynamicConfigManager.startup()
```

### DynamicConfigManager

配置更新时，会将数据顺序写到 `/config/changes` 路径的节点下，KafkaServer 启动时会初始化一个 `DynamicConfigManager` 对象，监听这个路径。

``` scala
class DynamicConfigManager(private val zkClient: KafkaZkClient,
                           private val configHandlers: Map[String, ConfigHandler],
                           private val changeExpirationMs: Long = 15*60*1000,
                           private val time: Time = Time.SYSTEM) extends Logging {


  private val configChangeListener = new ZkNodeChangeNotificationListener(zkClient, ConfigEntityChangeNotificationZNode.path,
    ConfigEntityChangeNotificationSequenceZNode.SequenceNumberPrefix, ConfigChangedNotificationHandler)

  /**
   * Begin watching for config changes
   */
  def startup(): Unit = {
    configChangeListener.init()

    // Apply all existing client/user configs to the ClientIdConfigHandler/UserConfigHandler to bootstrap the overrides
    configHandlers.foreach {
      case (ConfigType.User, handler) =>
        adminZkClient.fetchAllEntityConfigs(ConfigType.User).foreach {
          case (sanitizedUser, properties) => handler.processConfigChanges(sanitizedUser, properties)
        }
        adminZkClient.fetchAllChildEntityConfigs(ConfigType.User, ConfigType.Client).foreach {
          case (sanitizedUserClientId, properties) => handler.processConfigChanges(sanitizedUserClientId, properties)
        }
      case (configType, handler) =>
        adminZkClient.fetchAllEntityConfigs(configType).foreach {
          case (entityName, properties) => handler.processConfigChanges(entityName, properties)
        }
    }
  }
}
```

监听这个路径对应的处理 handler 为 `ConfigChangedNotificationHandler`，继承自 `NotificationHandler`，实现 `processNotifiction` 方法，处理新增的动态配置更新信息。

### ConfigHandler

`DynamicConfigManager` 持有 `configHandlers`，包含 4 中类型配置对应的 `ConfigHandler`

4 种类型配置的更新，对应 4 种 ConfigHandler

- BrokerConfigHandler
- TopicConfigHandler
- UserConfigHandler
- ClientIdConfighandler

#### BrokerConfigHandler

``` scala
/**
  * The BrokerConfigHandler will process individual broker config changes in ZK.
  * The callback provides the brokerId and the full properties set read from ZK.
  * This implementation reports the overrides to the respective ReplicationQuotaManager objects
  */
class BrokerConfigHandler(private val brokerConfig: KafkaConfig,
                          private val quotaManagers: QuotaManagers) extends ConfigHandler with Logging {

  def processConfigChanges(brokerId: String, properties: Properties): Unit = {
    def getOrDefault(prop: String): Long = {
      if (properties.containsKey(prop))
        properties.getProperty(prop).toLong
      else
        DefaultReplicationThrottledRate
    }
    if (brokerId == ConfigEntityName.Default) {
      brokerConfig.dynamicConfig.updateDefaultConfig(properties)
    } else if (brokerConfig.brokerId == brokerId.trim.toInt) {
      brokerConfig.dynamicConfig.updateBrokerConfig(brokerConfig.brokerId, properties)
      quotaManagers.leader.updateQuota(upperBound(getOrDefault(LeaderReplicationThrottledRateProp).toDouble))
      quotaManagers.follower.updateQuota(upperBound(getOrDefault(FollowerReplicationThrottledRateProp).toDouble))
      quotaManagers.alterLogDirs.updateQuota(upperBound(getOrDefault(ReplicaAlterLogDirsIoMaxBytesPerSecondProp).toDouble))
    }
  }
}
```

#### TopicConfigHandler

### Reconfigurable/BrokerReconfigurable

可以在运行中动态更新配置的实例都继承 `Reconfigurable`/`BrokerReconfigurable` 接口

- BrokerReconfigurable
    - DynamicThreadPool(KafkaServer)
        - numIoThreads
        - numNetworkThreads
        - numReplicaFetchers
        - numRecoveryThreadsPerDataDir
        - backgroundThreads
    - KafkaServer.logManager.cleaner
    - DynamicLogConfig
    - DynamicListenerConfig
    - kafkaServer.socketServer

