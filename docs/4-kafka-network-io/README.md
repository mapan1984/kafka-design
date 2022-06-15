# 4 kafka 对 nio 的包装

## NetworkSend

属性：
* destination 代表节点，与 KafkaChannel 的 id 一致
* buffers 发送的数据

作用：
* 修改原始发送的 ByteBuffer，将前 4 字节修改为 ByteBuffer 的大小，原始内容后移 4 字节
* 实现 `writeTo` 方法，将自身数据 buffers 写入 channel

发送 `NetworkSend` 时，可以根据 destination 找到对应的 `KafkaChannel`，

## NetworkReceive

属性：
* source 代表节点，与 KafkaChannel 的 id 一致
* size    存放接收数据前 4 byte，即此次数据的大小
* buffers 存放接收到的数据

作用：
* 实现 `readFrom` 方法，从 channel 中读数据到 buffers 中
* 从 ReadableByteChannel 读取，前 4 字节为当前整个响应的长度，根据长度读取内容到 buffer

## KafkaChannel

持有：
* id: 标识节点，同时 selector 可以根据 id 取到对应的 KafkaChannel
* Send(NetworkSend): 只能持有一个 send，write 发送当前 send 之后，会将 send 置为 null，之后可以设置下一个 send
* NetworkReceive: 只能持有一个 receive，read 接收完整数据后，将 receive 只为 null, 之后可以接收下一个 receive
* TransportLayer: send, receive 都通过 TransportLayer 转发给 SocketChannel

## TransportLayer

继承自 ScatteringByteChannel,GatheringByteChannel

持有 SelectionKey, SocketChannel

KafkaChannel 所有读写都经过 TransportLayer 转发给原来的 SocketChannel

TransportLayer 相比于原来的 Channel 主要作用是增加了 ssl 与 sasl 的支持

## Selector

可以监听多个连接，每个连接由 id 以及对应的 KafkaChannel 标识

### connect

connect 将 id 与远程连接的 KafkaChannel 对应起来

### send

发送 NetworkSend，根据 NetworkSend 的 destination 取到对应的 KafkaChannel，设置 KafkaChannel 的 send

### poll

nio select key，根据 key 找到对应的 KafkaChannel，执行 KafkaChannel 的 read, write 方法

## 补充知识

### interestOps

|                         |        |           |
|-------------------------|--------|-----------|
| SelectionKey.OP_READ    | `1<<0` | 0000 0001 |
| SelectionKey.OP_WRITE   | `1<<2` | 0000 0100 |
| SelectionKey.OP_CONNECT | `1<<3` | 0000 1000 |
| SelectionKey.OP_ACCEPT  | `1<<4` | 0001 0000 |
|                         |        | 0000 0000 |

取消 OP_CONNECT，增加 OP_READ

    key.interestOps(key.interestOps() & ~SelectionKey.OP_CONNECT | SelectionKey.OP_READ);

### ScatteringByteChannel,GatheringByteChannel

* scatteringbytechannel: 从通道读取数据分散到多个缓冲区 buffer，
* gatheringbytechannel: 将多个缓冲区 buffer 聚集起来写到通道

