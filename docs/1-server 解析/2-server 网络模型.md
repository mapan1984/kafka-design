# 1.2 server 网络模型

从上一节我们得知 kafka 服务启动的动作对应的代码是 `KafkaServer` 的 `startup()` 函数，在 `startup()` 函数中， 创建并启动了一系列组件，其中包括 `SocketServer`。

``` scala
// Create and start the socket server acceptor threads so that the bound port is known.
// Delay starting processors until the end of the initialization sequence to ensure
// that credentials have been loaded before processing authentications.
//
// Note that we allow the use of KRaft mode controller APIs when forwarding is enabled
// so that the Envelope request is exposed. This is only used in testing currently.
socketServer = new SocketServer(config, metrics, time, credentialProvider, apiVersionManager)
socketServer.startup(startProcessingRequests = false)
```

这一节解析 server 端的网络模型，即 server 端接收请求，处理并返回响应的过程。

对请求的监听正是通过 `SocketServer` 组件完成。

![](/kafka-design/images/1-server/kafka server network.png)

## SocketServer

`SocketServer` 支持 2 种类型的请求：

1. 数据(data plane)：
    - 处理来自 client 或者同集群其他 broker 的请求
    - 线程模型是：
        - 可以配置多个 listener，每个 listener 对应 1 个 Acceptor 线程，处理新的连接请求
        - Acceptor 有 N 个 Processor 线程，Processor 有自己的 selector 用来读来自 sockets 的请求(`num.network.threads`)
        - M 个 Handler 线程处理请求并产生响应发回给 Processor 线程(`num.io.threads`)
2. 控制(control plane)：
    - 处理来自 controller 的请求。这是可选的并且通过指定 `control.plane.listener.name` 进行配置。如果没有配置，controller 的请求也会像数据请求一样通过 listener 处理
    - 线程模型是：
        - 1 个 Acceptor 线程处理新的连接请求
        - Acceptor 有 1 个 Processor 线程，Processor 有自己的 selector 用来读来自 socket 的请求
        - 1 个 Handler 线程处理请求并产生响应发回给 Processor 线程

查看 [SocketServer](/kafka-design/6-src/core/kafka/network/SocketServer) 代码，

`SocketServer` 构造时会初始以下关键属性：

``` scala
// data-plane
private val dataPlaneProcessors = new ConcurrentHashMap[Int, Processor]()
private[network] val dataPlaneAcceptors = new ConcurrentHashMap[EndPoint, Acceptor]()
val dataPlaneRequestChannel = new RequestChannel(maxQueuedRequests, DataPlaneMetricPrefix, time)
// control-plane
private var controlPlaneProcessorOpt : Option[Processor] = None
private[network] var controlPlaneAcceptorOpt : Option[Acceptor] = None
val controlPlaneRequestChannelOpt: Option[RequestChannel] = config.controlPlaneListenerName.map(_ =>
  new RequestChannel(20, ControlPlaneMetricPrefix, time))
```

`SocketServer` 的 `startup()` 函数中主要动作为：

``` scala
this.synchronized {
  connectionQuotas = new ConnectionQuotas(config, time, metrics)
  // 1. 创建处理 controller 请求的 Acceptor, Processor
  createControlPlaneAcceptorAndProcessor(config.controlPlaneListener)
  // 2. 创建处理数据请求的 Acceptor, Processor
  createDataPlaneAcceptorsAndProcessors(config.numNetworkThreads, config.dataPlaneListeners)
  if (startProcessingRequests) {
    // 3. 启动线程 kafkaThread，后台运行 Acceptor 与 Processor
    this.startProcessingRequests()
  }
}
```

主要有 3 步：

1. 创建处理 controller 请求的 Acceptor, Processor
2. 创建处理数据请求的 Acceptor, Processor
3. 启动线程 kafkaThread，后台运行 Acceptor 与 Processor

``` scala
private def createDataPlaneAcceptorsAndProcessors(dataProcessorsPerListener: Int,
                                                  endpoints: Seq[EndPoint]): Unit = {
  // 对每个 endpoint
  endpoints.foreach { endpoint =>
    connectionQuotas.addListener(config, endpoint.listenerName)
    // 创建 Acceptor
    val dataPlaneAcceptor = createAcceptor(endpoint, DataPlaneMetricPrefix)
    // 1. 创建 dataProcessorsPerListener 个 Processor
    // 2. 将 Processor 加入到 dataPlaneRequestChannel 的 processors 中
    // 3. 将 Processor 加入到 SocketServer 的 dataPlaneProcessors 中
    // 4. 将 Processor 加入到 Acceptor 的 processors 中
    addDataPlaneProcessors(dataPlaneAcceptor, endpoint, dataProcessorsPerListener)
    // 将 Acceptor 加入到 SocketServer 的 dataPlaneAcceptors 中
    dataPlaneAcceptors.put(endpoint, dataPlaneAcceptor)
    info(s"Created data-plane acceptor and processors for endpoint : ${endpoint.listenerName}")
  }
}
```

``` scala
private def createControlPlaneAcceptorAndProcessor(endpointOpt: Option[EndPoint]): Unit = {
  endpointOpt.foreach { endpoint =>
    connectionQuotas.addListener(config, endpoint.listenerName)
    val controlPlaneAcceptor = createAcceptor(endpoint, ControlPlaneMetricPrefix)
    val controlPlaneProcessor = newProcessor(nextProcessorId, controlPlaneRequestChannelOpt.get, connectionQuotas, endpoint.listenerName, endpoint.securityProtocol, memoryPool)
    controlPlaneAcceptorOpt = Some(controlPlaneAcceptor)
    controlPlaneProcessorOpt = Some(controlPlaneProcessor)
    val listenerProcessors = new ArrayBuffer[Processor]()
    listenerProcessors += controlPlaneProcessor
    controlPlaneRequestChannelOpt.foreach(_.addProcessor(controlPlaneProcessor))
    nextProcessorId += 1
    controlPlaneAcceptor.addProcessors(listenerProcessors, ControlPlaneThreadPrefix)
    info(s"Created control-plane acceptor and processor for endpoint : ${endpoint.listenerName}")
  }
}
```

这里做的工作为：

1. 为每个 endpoint 创建 1 个 `Acceptor`，`Acceptor` 负责监听请求
2. 对每个 endpoint 创建 N (`num.network.threads`) 个 `Processor`，同时：
    1. 将 `Processor` 记录到 `RequestChannel` 的 `processors` hash map 中
    2. 将 `Processor` 记录到 `SocketServer` 的 `dataPlaneProcessors` hash map 中
    3. 将 `Processor` 加入到 `Acceptor` 的 `processors` 数组中
3. 将 `Acceptor` 加入到 `SocketServer` 的 `dataPlaneAcceptors` hash map 中

这里完成之后，处理请求所需的 `SocketServer`, `Acceptor`, `Processor`, `RequestChannel` 就都创建完成了，下面主要看一下 `Acceptor`, `Processor`, `RequestChannel` 各自的作用。

## Acceptor

[Acceptor](/kafka-design/6-src/core/kafka/network/SocketServer/Acceptor/)

## Processor

[Processor](/kafka-design/6-src/core/kafka/network/SocketServer/Processor/)

## RequestChannel

[RequestChannel](/kafka-design/6-src/core/kafka/network/RequestChannel/)

## KafkaRequestHandler

[KafkaRequestHandler](/kafka-design/6-src/core/kafka/server/KafkaRequestHandler/)

## 监控

### requestHandlerAvgIdlePercent

`Meter` 类型，在 `KafkaRequestHandlerPool` 中创建，创建每个 `KafkaRequestHandler` 时传入

``` scala
/* a meter to track the average free capacity of the request handlers */
private val aggregateIdleMeter = newMeter(requestHandlerAvgIdleMetricName, "percent", TimeUnit.NANOSECONDS)


val runnables = new mutable.ArrayBuffer[KafkaRequestHandler](numThreads)
for (i <- 0 until numThreads) {
  createHandler(i)
}

def createHandler(id: Int): Unit = synchronized {
  runnables += new KafkaRequestHandler(id, brokerId, aggregateIdleMeter, threadPoolSize, requestChannel, apis, time)
  KafkaThread.daemon(logAndThreadNamePrefix + "-kafka-request-handler-" + id, runnables(id)).start()
}
```

`KafkaRequestHandler` 作为后台线程启动，启动后不同从 `RequestChannel` 中获取请求，`Meter.mark()` 时，以获取请求的耗时除以线程总数。


``` scala
  def run(): Unit = {
    while (!stopped) {
      // We use a single meter for aggregate idle percentage for the thread pool.
      // Since meter is calculated as total_recorded_value / time_window and
      // time_window is independent of the number of threads, each recorded idle
      // time should be discounted by # threads.
      val startSelectTime = time.nanoseconds

      val req = requestChannel.receiveRequest(300)
      val endTime = time.nanoseconds
      val idleTime = endTime - startSelectTime

      // NOTE: 从 RequestChannel 中获取请求的时间，是 KafkaRequestHandler 空闲的时间
      aggregateIdleMeter.mark(idleTime / totalHandlerThreads.get)

      // NOTE: 处理请求的时间为 KafkaRequestHandler 非空闲的时间
      req match {
        case RequestChannel.ShutdownRequest =>
          debug(s"Kafka request handler $id on broker $brokerId received shut down command")
          shutdownComplete.countDown()
          return

        case request: RequestChannel.Request =>
          try {
            request.requestDequeueTimeNanos = endTime
            trace(s"Kafka request handler $id on broker $brokerId handling request $request")
            apis.handle(request)
          } catch {
            case e: FatalExitError =>
              shutdownComplete.countDown()
              Exit.exit(e.statusCode)
            case e: Throwable => error("Exception when handling request", e)
          } finally {
            request.releaseBuffer()
          }

        case null => // continue
      }
    }
    shutdownComplete.countDown()
  }
```

requestHandler 线程不断循环做 2 件事：

1. 从  channel 中获取待处理请求
2. 处理请求（读/写请求/其他）

假设每次循环，第 1 件事耗时 A，第 2 件事耗时 B

空闲率 = A / (A + B)

所以空闲率增加，可能的原因是请求数减少，导致 A 增加；或者读/写磁盘更快，处理请求耗时减少

### NetworkProcessorAvgIdlePercent

`Gauge` 类型，在 `SocketServer.startup()` 时定义

``` scala
    newGauge(s"${DataPlaneMetricPrefix}NetworkProcessorAvgIdlePercent", () => SocketServer.this.synchronized {
      val ioWaitRatioMetricNames = dataPlaneProcessors.values.asScala.iterator.map { p =>
        metrics.metricName("io-wait-ratio", MetricsGroup, p.metricTags)
      }
      ioWaitRatioMetricNames.map { metricName =>
        Option(metrics.metric(metricName)).fold(0.0)(m => Math.min(m.metricValue.asInstanceOf[Double], 1.0))
      }.sum / dataPlaneProcessors.size
    })
```

计算方法是取得每个 networkProcessor 的 io-wait-ratio，除以 networkProcessor 总数。io-wait 的比例，即是 networkProcessor 空闲的占比。

io-wait-ratio 指标在 `Selector` 中计算，`Meter` 类型

