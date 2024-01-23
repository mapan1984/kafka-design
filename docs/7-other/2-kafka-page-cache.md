# page cache

1. producer 生产消息时，会使用 `pwrite()` 系统调用(对应 Java NIO `FileChannel.write()` API)按偏移量写入数据，并且都会先写入 page cache 里
2. consumer 消费消息时，会使用 `sendfile()` 系统调用(对应 Java NIO `FileChannel.transferTo()` API)，**零拷贝**地将数据从 page cache 传输到 broker 的 socket buffer，再通过网络传输
3. follower 同步消息，与 consumer 同理
4. page cache 中的数据会随着内核中 flusher 线程的调度以及对 `sync()/fsync()` 的调用写回到磁盘，就算进程崩溃，也不用担心数据丢失。

另外，如果 consumer 要消费的消息不在 page cache 里，才会去磁盘读取，并且会顺便预读出一些相邻的块放入 page cache，以方便下一次读取。

由此我们可以得出重要的结论：如果 Kafka producer 的生产速率与 consumer 的消费速率相差不大，那么就能几乎只靠对 broker page cache 的读写完成整个生产-消费过程，磁盘访问非常少。并且 Kafka 持久化消息到各个 topic 的 partition 文件时，是只追加的顺序写，充分利用了磁盘顺序访问快的特性，效率高。

## 参考

- https://cloud.tencent.com/developer/article/1488144
- https://www.cnblogs.com/xiaolincoding/p/13719610.html
