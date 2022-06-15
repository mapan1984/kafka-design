# ControllerEventProcessor

处理 `ControllerEvent`

``` scala
trait ControllerEventProcessor {
  def process(event: ControllerEvent): Unit
  def preempt(event: ControllerEvent): Unit
}
```
