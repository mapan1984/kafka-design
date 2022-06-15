## consumer 什么时候被认为下线

1. 当 subscribe 完成并调用 poll() 之后，consumer 会自动加入 group，并周期性的向 server 发送 heartbeat，当 server 距上一次收到 heartbeat 的时间超过 `session.timeout.ms` 后，会认为 consumer dead
2. 另一种情况是 consumer 在持续发送 heartbeat，但是没有实际的消费行为，当距上一次调用 poll() 的时间超过 `max.poll.interval.ms` 后，会将 consumer 移出 group。这种情况下 consumer 可能会在调用 commitSync() 时出现 offset commit failure，这是正常的，为了保证只接受来自活跃 consumer 的 offset。


consumer 提供 2 个参数来控制 poll loop 的行为：

- `max.poll.interval.ms`: 通过提高这个参数，可以给 consumer 更多的时间处理单次 poll 拉取的数据。
- `max.poll.records`: 限制单次 poll 调用拉取的记录数。

对于消息处理用时无法预测的用户场景，上面 2 个参数是不够的。这种情况下推荐的方式将处理消息的步骤放到单独的线程，确保 consumer 可以持续的调用 poll，关闭自动 offset commit，在消息处理完之后调用 commitSync/commitAsync。

自动提交 offset：

- `enable.auto.commit`
- `auto.commit.interval.ms`

手动提交 offset 的 2 种方式：

- `commitSync`
- `commitAsync`

订阅 topic 的 2 种方式：

- `subscribe`: topic 粒度，具体的 partition 分配由消费组协调者分配
- `assign`: topic-partition 粒度，可以由自己分配 partition

## other

- `offsets.retention.minutes`: 消费组记录保留时间
