# ControllerEventThread

拉取 `ControllerEventManager` 中的事件，进行处理

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
