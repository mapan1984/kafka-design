# KafkaServer

## 方法

### startup

``` scala
/**
 * Start up API for bringing up a single instance of the Kafka server.
 * Instantiates the LogManager, the SocketServer and the request handlers - KafkaRequestHandlers
 */
override def startup(): Unit = {
  try {
    info("starting")

    if (isShuttingDown.get)
      throw new IllegalStateException("Kafka server is still shutting down, cannot re-start!")

    if (startupComplete.get)
      return

    val canStartup = isStartingUp.compareAndSet(false, true)
    if (canStartup) {
      brokerState.set(BrokerState.STARTING)

      /* setup zookeeper */
      initZkClient(time)
      configRepository = new ZkConfigRepository(new AdminZkClient(zkClient))

      /* initialize features */
      _featureChangeListener = new FinalizedFeatureChangeListener(featureCache, _zkClient)
      if (config.isFeatureVersioningSupported) {
        _featureChangeListener.initOrThrow(config.zkConnectionTimeoutMs)
      }

      /* Get or create cluster_id */
      _clusterId = getOrGenerateClusterId(zkClient)
      info(s"Cluster ID = ${clusterId}")

      /* load metadata */
      val (preloadedBrokerMetadataCheckpoint, initialOfflineDirs) =
        BrokerMetadataCheckpoint.getBrokerMetadataAndOfflineDirs(config.logDirs, ignoreMissing = true)

      if (preloadedBrokerMetadataCheckpoint.version != 0) {
        throw new RuntimeException(s"Found unexpected version in loaded `meta.properties`: " +
          s"$preloadedBrokerMetadataCheckpoint. Zk-based brokers only support version 0 " +
          "(which is implicit when the `version` field is missing).")
      }

      /* check cluster id */
      if (preloadedBrokerMetadataCheckpoint.clusterId.isDefined && preloadedBrokerMetadataCheckpoint.clusterId.get != clusterId)
        throw new InconsistentClusterIdException(
          s"The Cluster ID ${clusterId} doesn't match stored clusterId ${preloadedBrokerMetadataCheckpoint.clusterId} in meta.properties. " +
          s"The broker is trying to join the wrong cluster. Configured zookeeper.connect may be wrong.")

      /* generate brokerId */
      config.brokerId = getOrGenerateBrokerId(preloadedBrokerMetadataCheckpoint)
      logContext = new LogContext(s"[KafkaServer id=${config.brokerId}] ")
      this.logIdent = logContext.logPrefix

      // initialize dynamic broker configs from ZooKeeper. Any updates made after this will be
      // applied after DynamicConfigManager starts.
      config.dynamicConfig.initialize(zkClient)

      /* start scheduler */
      kafkaScheduler = new KafkaScheduler(config.backgroundThreads)
      kafkaScheduler.startup()

      /* create and configure metrics */
      kafkaYammerMetrics = KafkaYammerMetrics.INSTANCE
      kafkaYammerMetrics.configure(config.originals)
      metrics = Server.initializeMetrics(config, time, clusterId)

      /* register broker metrics */
      _brokerTopicStats = new BrokerTopicStats

      quotaManagers = QuotaFactory.instantiate(config, metrics, time, threadNamePrefix.getOrElse(""))
      KafkaBroker.notifyClusterListeners(clusterId, kafkaMetricsReporters ++ metrics.reporters.asScala)

      logDirFailureChannel = new LogDirFailureChannel(config.logDirs.size)

      /* start log manager */
      logManager = LogManager(config, initialOfflineDirs,
        new ZkConfigRepository(new AdminZkClient(zkClient)),
        kafkaScheduler, time, brokerTopicStats, logDirFailureChannel, config.usesTopicId)
      brokerState.set(BrokerState.RECOVERY)
      logManager.startup(zkClient.getAllTopicsInCluster())

      metadataCache = MetadataCache.zkMetadataCache(config.brokerId)
      // Enable delegation token cache for all SCRAM mechanisms to simplify dynamic update.
      // This keeps the cache up-to-date if new SCRAM mechanisms are enabled dynamically.
      tokenCache = new DelegationTokenCache(ScramMechanism.mechanismNames)
      credentialProvider = new CredentialProvider(ScramMechanism.mechanismNames, tokenCache)

      if (enableForwarding) {
        this.forwardingManager = Some(ForwardingManager(
          config,
          metadataCache,
          time,
          metrics,
          threadNamePrefix
        ))
        forwardingManager.foreach(_.start())
      }

      val apiVersionManager = ApiVersionManager(
        ListenerType.ZK_BROKER,
        config,
        forwardingManager,
        brokerFeatures,
        featureCache
      )

      // Create and start the socket server acceptor threads so that the bound port is known.
      // Delay starting processors until the end of the initialization sequence to ensure
      // that credentials have been loaded before processing authentications.
      //
      // Note that we allow the use of KRaft mode controller APIs when forwarding is enabled
      // so that the Envelope request is exposed. This is only used in testing currently.
      socketServer = new SocketServer(config, metrics, time, credentialProvider, apiVersionManager)
      socketServer.startup(startProcessingRequests = false)

      /* start replica manager */
      alterIsrManager = if (config.interBrokerProtocolVersion.isAlterIsrSupported) {
        AlterIsrManager(
          config = config,
          metadataCache = metadataCache,
          scheduler = kafkaScheduler,
          time = time,
          metrics = metrics,
          threadNamePrefix = threadNamePrefix,
          brokerEpochSupplier = () => kafkaController.brokerEpoch,
          config.brokerId
        )
      } else {
        AlterIsrManager(kafkaScheduler, time, zkClient)
      }
      alterIsrManager.start()

      replicaManager = createReplicaManager(isShuttingDown)
      replicaManager.startup()

      val brokerInfo = createBrokerInfo
      val brokerEpoch = zkClient.registerBroker(brokerInfo)

      // Now that the broker is successfully registered, checkpoint its metadata
      checkpointBrokerMetadata(ZkMetaProperties(clusterId, config.brokerId))

      /* start token manager */
      tokenManager = new DelegationTokenManager(config, tokenCache, time , zkClient)
      tokenManager.startup()

      /* start kafka controller */
      kafkaController = new KafkaController(config, zkClient, time, metrics, brokerInfo, brokerEpoch, tokenManager, brokerFeatures, featureCache, threadNamePrefix)
      kafkaController.startup()

      adminManager = new ZkAdminManager(config, metrics, metadataCache, zkClient)

      /* start group coordinator */
      // Hardcode Time.SYSTEM for now as some Streams tests fail otherwise, it would be good to fix the underlying issue
      groupCoordinator = GroupCoordinator(config, replicaManager, Time.SYSTEM, metrics)
      groupCoordinator.startup(() => zkClient.getTopicPartitionCount(Topic.GROUP_METADATA_TOPIC_NAME).getOrElse(config.offsetsTopicPartitions))

      /* start transaction coordinator, with a separate background thread scheduler for transaction expiration and log loading */
      // Hardcode Time.SYSTEM for now as some Streams tests fail otherwise, it would be good to fix the underlying issue
      transactionCoordinator = TransactionCoordinator(config, replicaManager, new KafkaScheduler(threads = 1, threadNamePrefix = "transaction-log-manager-"),
        () => new ProducerIdManager(config.brokerId, zkClient), metrics, metadataCache, Time.SYSTEM)
      transactionCoordinator.startup(
        () => zkClient.getTopicPartitionCount(Topic.TRANSACTION_STATE_TOPIC_NAME).getOrElse(config.transactionTopicPartitions))

      /* start auto topic creation manager */
      this.autoTopicCreationManager = AutoTopicCreationManager(
        config,
        metadataCache,
        time,
        metrics,
        threadNamePrefix,
        Some(adminManager),
        Some(kafkaController),
        groupCoordinator,
        transactionCoordinator,
        enableForwarding
      )
      autoTopicCreationManager.start()

      /* Get the authorizer and initialize it if one is specified.*/
      authorizer = config.authorizer
      authorizer.foreach(_.configure(config.originals))
      val authorizerFutures: Map[Endpoint, CompletableFuture[Void]] = authorizer match {
        case Some(authZ) =>
          authZ.start(brokerInfo.broker.toServerInfo(clusterId, config)).asScala.map { case (ep, cs) =>
            ep -> cs.toCompletableFuture
          }
        case None =>
          brokerInfo.broker.endPoints.map { ep =>
            ep.toJava -> CompletableFuture.completedFuture[Void](null)
          }.toMap
      }

      val fetchManager = new FetchManager(Time.SYSTEM,
        new FetchSessionCache(config.maxIncrementalFetchSessionCacheSlots,
          KafkaServer.MIN_INCREMENTAL_FETCH_SESSION_EVICTION_MS))

      /* start processing requests */
      val zkSupport = ZkSupport(adminManager, kafkaController, zkClient, forwardingManager, metadataCache)
      dataPlaneRequestProcessor = new KafkaApis(socketServer.dataPlaneRequestChannel, zkSupport, replicaManager, groupCoordinator, transactionCoordinator,
        autoTopicCreationManager, config.brokerId, config, configRepository, metadataCache, metrics, authorizer, quotaManagers,
        fetchManager, brokerTopicStats, clusterId, time, tokenManager, apiVersionManager)

      dataPlaneRequestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.dataPlaneRequestChannel, dataPlaneRequestProcessor, time,
        config.numIoThreads, s"${SocketServer.DataPlaneMetricPrefix}RequestHandlerAvgIdlePercent", SocketServer.DataPlaneThreadPrefix)

      socketServer.controlPlaneRequestChannelOpt.foreach { controlPlaneRequestChannel =>
        controlPlaneRequestProcessor = new KafkaApis(controlPlaneRequestChannel, zkSupport, replicaManager, groupCoordinator, transactionCoordinator,
          autoTopicCreationManager, config.brokerId, config, configRepository, metadataCache, metrics, authorizer, quotaManagers,
          fetchManager, brokerTopicStats, clusterId, time, tokenManager, apiVersionManager)

        controlPlaneRequestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.controlPlaneRequestChannelOpt.get, controlPlaneRequestProcessor, time,
          1, s"${SocketServer.ControlPlaneMetricPrefix}RequestHandlerAvgIdlePercent", SocketServer.ControlPlaneThreadPrefix)
      }

      Mx4jLoader.maybeLoad()

      /* Add all reconfigurables for config change notification before starting config handlers */
      config.dynamicConfig.addReconfigurables(this)

      /* start dynamic config manager */
      dynamicConfigHandlers = Map[String, ConfigHandler](ConfigType.Topic -> new TopicConfigHandler(logManager, config, quotaManagers, kafkaController),
                                                         ConfigType.Client -> new ClientIdConfigHandler(quotaManagers),
                                                         ConfigType.User -> new UserConfigHandler(quotaManagers, credentialProvider),
                                                         ConfigType.Broker -> new BrokerConfigHandler(config, quotaManagers),
                                                         ConfigType.Ip -> new IpConfigHandler(socketServer.connectionQuotas))

      // Create the config manager. start listening to notifications
      dynamicConfigManager = new DynamicConfigManager(zkClient, dynamicConfigHandlers)
      dynamicConfigManager.startup()

      socketServer.startProcessingRequests(authorizerFutures)

      brokerState.set(BrokerState.RUNNING)
      shutdownLatch = new CountDownLatch(1)
      startupComplete.set(true)
      isStartingUp.set(false)
      AppInfoParser.registerAppInfo(Server.MetricsPrefix, config.brokerId.toString, metrics, time.milliseconds())
      info("started")
    }
  }
  catch {
    case e: Throwable =>
      fatal("Fatal error during KafkaServer startup. Prepare to shutdown", e)
      isStartingUp.set(false)
      shutdown()
      throw e
  }
}
```

创建 `SocketServer` 并调用 `socketServer.startup()`
