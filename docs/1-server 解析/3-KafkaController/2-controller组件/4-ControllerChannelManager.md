# 1.3.2.4 ControllerChannelManager

Controller 会向 broker 发送 3 种类型的请求：

- LeaderAndIsrRequest
- UpdateMetadataRequest
- StopReplicaRequest

ControllerChannelManager 维护与 broker 之间的连接

``` scala
protected val brokerStateInfo = new HashMap[Int, ControllerBrokerStateInfo]
private val brokerLock = new Object
```

ControllerBrokerStateInfo 定义如下：

``` scala
case class ControllerBrokerStateInfo(networkClient: NetworkClient,
                                     brokerNode: Node,
                                     messageQueue: BlockingQueue[QueueItem],
                                     requestSendThread: RequestSendThread,
                                     queueSizeGauge: Gauge[Int],
                                     requestRateAndTimeMetrics: Timer,
                                     reconfigurableChannelBuilder: Option[Reconfigurable])
```
