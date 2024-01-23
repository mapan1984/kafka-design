# 1.8 LogSegment

`append()`

* 写消息到文件
* 增加自身维护的 offsetIndex, timeIndex

`flush()`

* log.flush()
* offsetIndex.flush()
* timeIndex.flush()
* txnIndex.flush()

nio
* log 文件，`java.nio.channels.WritableByteChannel.write()`
* index 文件，`java.nio.MappedByteBuffer` 直接对应一个文件，方法 `force()` 同步内存内容到文件
