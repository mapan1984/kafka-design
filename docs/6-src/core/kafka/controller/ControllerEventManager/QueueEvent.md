# QueueEvent

``` scala
class QueuedEvent(val event: ControllerEvent,
                  val enqueueTimeMs: Long) {
  val processingStarted = new CountDownLatch(1)
  val spent = new AtomicBoolean(false)

  def process(processor: ControllerEventProcessor): Unit = {
    if (spent.getAndSet(true))
      return
    processingStarted.countDown()
    processor.process(event)
  }

  def preempt(processor: ControllerEventProcessor): Unit = {
    if (spent.getAndSet(true))
      return
    processor.preempt(event)
  }

  def awaitProcessing(): Unit = {
    processingStarted.await()
  }

  override def toString: String = {
    s"QueuedEvent(event=$event, enqueueTimeMs=$enqueueTimeMs)"
  }
}
```
