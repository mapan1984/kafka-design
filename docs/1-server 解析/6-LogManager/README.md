# 1.6 LogManager

每个 broker 进程有 1 个 LogManager 实例，LogManager 负责管理这个 broker 所有 topic partition 对应的 Log。

每个 topic partition 对应一个 Log 实例，Log 实例中包含多个 LogSegment。

`LogManager` 负责：

* log creation
* log retrieval
* log cleaning

所有 read 和 write 操作都会被代理给单独的 `Log` 对象负责

`LogManager` 维护一个或多个目录，新的 log 会在 log 最少的目录中创建

一个后台线程通过周期回收 log segment 处理 log retention

