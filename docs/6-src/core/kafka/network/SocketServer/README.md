# SocketServer

支持 2 种类型的请求：
1. 数据：
    - 处理来自 client 或者同集群其他 broker 的请求
    - 线程模型是：
        - 可以配置多个 listener，每个 listener 对应 1 个 Acceptor 线程，处理新的连接请求
        - Acceptor 有 N 个 Processor 线程，Processor 有自己的 selector 用来读来自 sockets 的请求
        - M 个 Handler 线程处理请求并产生响应发回给 Processor 线程
2. 控制：
    - 处理来自 controller 的请求。这是可选的并且通过指定 `control.plane.listener.name` 进行配置。如果没有配置，controller 的请求也会像数据请求一样通过 listener 处理
    - 线程模型是：
        - 1 个 Acceptor 线程处理新的连接请求
        - Acceptor 有 1 个 Processor 线程，Processor 有自己的 selector 用来读来自 socket 的请求
        - 1 个 Handler 线程处理请求并产生响应发回给 Processor 线程

## 属性

``` scala
private val maxQueuedRequests = config.queuedMaxRequests

private val logContext = new LogContext(s"[SocketServer brokerId=${config.brokerId}] ")
this.logIdent = logContext.logPrefix

private val memoryPoolSensor = metrics.sensor("MemoryPoolUtilization")
private val memoryPoolDepletedPercentMetricName = metrics.metricName("MemoryPoolAvgDepletedPercent", MetricsGroup)
private val memoryPoolDepletedTimeMetricName = metrics.metricName("MemoryPoolDepletedTimeTotal", MetricsGroup)
memoryPoolSensor.add(new Meter(TimeUnit.MILLISECONDS, memoryPoolDepletedPercentMetricName, memoryPoolDepletedTimeMetricName))
private val memoryPool = if (config.queuedMaxBytes > 0) new SimpleMemoryPool(config.queuedMaxBytes, config.socketRequestMaxBytes, false, memoryPoolSensor) else MemoryPool.NONE
// data-plane
// Processor 记录
private val dataPlaneProcessors = new ConcurrentHashMap[Int, Processor]()
private[network] val dataPlaneAcceptors = new ConcurrentHashMap[EndPoint, Acceptor]()
val dataPlaneRequestChannel = new RequestChannel(maxQueuedRequests, DataPlaneMetricPrefix, time)
// control-plane
private var controlPlaneProcessorOpt : Option[Processor] = None
private[network] var controlPlaneAcceptorOpt : Option[Acceptor] = None
val controlPlaneRequestChannelOpt: Option[RequestChannel] = config.controlPlaneListenerName.map(_ =>
  new RequestChannel(20, ControlPlaneMetricPrefix, time))

private var nextProcessorId = 0
private var connectionQuotas: ConnectionQuotas = _
private var startedProcessingRequests = false
private var stoppedProcessingRequests = false
```

