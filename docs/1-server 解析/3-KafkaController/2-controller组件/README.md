# 1.3.2 controller 组件

## ControllerChannelManager

## 事件处理

### ControllerEventProcessor

### QueuedEvent

### ControllerEventManager


维持一个 `queue` 存放事件：

``` scala
private val queue = new LinkedBlockingQueue[QueuedEvent]
```

同时会创建一个 `thread` 在后台不断从 `queue` 中取事件并进行处理


``` scala
private[controller] var thread = new ControllerEventThread(ControllerEventThreadName)
```

``` scala
class ControllerEventThread(name: String) extends ShutdownableThread(name = name, isInterruptible = false) {
  logIdent = s"[ControllerEventThread controllerId=$controllerId] "

  override def doWork(): Unit = {
    val dequeued = pollFromEventQueue()
    dequeued.event match {
      case ShutdownEventThread => // The shutting down of the thread has been initiated at this point. Ignore this event.
      case controllerEvent =>
        _state = controllerEvent.state

        eventQueueTimeHist.update(time.milliseconds() - dequeued.enqueueTimeMs)

        try {
          def process(): Unit = dequeued.process(processor)

          rateAndTimeMetrics.get(state) match {
            case Some(timer) => timer.time { process() }
            case None => process()
          }
        } catch {
          case e: Throwable => error(s"Uncaught error processing event $controllerEvent", e)
        }

        _state = ControllerState.Idle
    }
  }
}
```

`KafkaController` 继承自 `ControllerEventProcessor`，定义 `process()` 方法如下：

``` scala
override def process(event: ControllerEvent): Unit = {
  try {
    event match {
      case event: MockEvent =>
        // Used only in test cases
        event.process()
      case ShutdownEventThread =>
        error("Received a ShutdownEventThread event. This type of event is supposed to be handle by ControllerEventThread")
      case AutoPreferredReplicaLeaderElection =>
        processAutoPreferredReplicaLeaderElection()
      case ReplicaLeaderElection(partitions, electionType, electionTrigger, callback) =>
        processReplicaLeaderElection(partitions, electionType, electionTrigger, callback)
      case UncleanLeaderElectionEnable =>
        processUncleanLeaderElectionEnable()
      case TopicUncleanLeaderElectionEnable(topic) =>
        processTopicUncleanLeaderElectionEnable(topic)
      case ControlledShutdown(id, brokerEpoch, callback) =>
        processControlledShutdown(id, brokerEpoch, callback)
      case LeaderAndIsrResponseReceived(response, brokerId) =>
        processLeaderAndIsrResponseReceived(response, brokerId)
      case UpdateMetadataResponseReceived(response, brokerId) =>
        processUpdateMetadataResponseReceived(response, brokerId)
      case TopicDeletionStopReplicaResponseReceived(replicaId, requestError, partitionErrors) =>
        processTopicDeletionStopReplicaResponseReceived(replicaId, requestError, partitionErrors)
      case BrokerChange =>
        processBrokerChange()
      case BrokerModifications(brokerId) =>
        processBrokerModification(brokerId)
      case ControllerChange =>
        processControllerChange()
      case Reelect =>
        processReelect()
      case RegisterBrokerAndReelect =>
        processRegisterBrokerAndReelect()
      case Expire =>
        processExpire()
      case TopicChange =>
        processTopicChange()
      case LogDirEventNotification =>
        processLogDirEventNotification()
      case PartitionModifications(topic) =>
        processPartitionModifications(topic)
      case TopicDeletion =>
        processTopicDeletion()
      case ApiPartitionReassignment(reassignments, callback) =>
        processApiPartitionReassignment(reassignments, callback)
      case ZkPartitionReassignment =>
        processZkPartitionReassignment()
      case ListPartitionReassignments(partitions, callback) =>
        processListPartitionReassignments(partitions, callback)
      case UpdateFeatures(request, callback) =>
        processFeatureUpdates(request, callback)
      case PartitionReassignmentIsrChange(partition) =>
        processPartitionReassignmentIsrChange(partition)
      case IsrChangeNotification =>
        processIsrChangeNotification()
      case AlterIsrReceived(brokerId, brokerEpoch, isrsToAlter, callback) =>
        processAlterIsr(brokerId, brokerEpoch, isrsToAlter, callback)
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

这里会对事件进行处理
