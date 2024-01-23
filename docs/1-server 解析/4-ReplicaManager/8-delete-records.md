# delete records

需要将请求发送到 partition leader 所在的 broker 上，并且等待删除操作同步到其他 partition follower 上。

通过 `maybeIncrementLogStartOffset()` 增加 log start offset
