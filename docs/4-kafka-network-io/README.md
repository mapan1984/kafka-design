# 4 kafka 对 nio 的包装

## NetworkSend

属性：

* `destination`: 代表目标节点，与 KafkaChannel 的 id 一致
* `buffers`: 发送的数据

作用：

* 修改原始发送的 ByteBuffer，将前 4 字节修改为 ByteBuffer 的大小，原始内容后移 4 字节
* 实现 `writeTo` 方法，将自身数据 buffers 写入 java nio channel，返回写入字节数

发送 `NetworkSend` 时，可以根据 destination 找到对应的 `KafkaChannel`，

## NetworkReceive

属性：

* `source`: 代表来源节点，与 KafkaChannel 的 id 一致
* `size`: 存放接收数据前 4 byte，即此次数据的大小
* `buffer`: 存放接收到的数据

作用：

* 实现 `readFrom` 方法，从 java nio channel 中读数据到 `buffer` 中
    * 首先读取前 4 字节数据到 `size`，前 4 字节为当前整个响应大小
    * 得到响应大小后，初始化 `buffer`，并将后续响应数据读取到 `buffer`

## KafkaChannel

channel 表示一个具体的 TCP 连接

属性：

* `id`: 表示 channel 对应的节点。selector 可以根据 id 取到对应的 KafkaChannel

* `Send(NetworkSend)`: 只能持有一个 send，write 发送当前 send 之后，会将 send 置为 null，之后可以设置下一个 send
* `NetworkReceive`: 只能持有一个 receive，read 接收完整数据后，将 receive 置为 null, 之后可以接收下一个 receive

* `TransportLayer`: send, receive 都通过 TransportLayer 转发给 SocketChannel
* `Authenticator`: 提供对连接的认证支持，向 TransportLayer 转发读写

## TransportLayer

继承自 ScatteringByteChannel,GatheringByteChannel

持有 `SelectionKey`, `SocketChannel`

KafkaChannel 所有读写都经过 TransportLayer 转发给原来的 SocketChannel

TransportLayer 相比于原来的 Channel 主要作用是可以增加了 ssl:

* `PlaintextTransportLayer`：直接转发请求给原来的 `SocketChannel`
* `SslTransportLayer`：实现 handshake 方法完成 ssl 握手，read, write 方法需要通过 sslEngine 对数据进行对应的 unwrap 与 wrap

## Authenticator

Authenticator 提供 `authenticate()` 方法完成对连接的认证，提供 `principal()` 方法获取这个连接的用户身份

* `PlaintextAuthenticator`: `authenticate()` 方法为空，`principal()` 方法返回匿名用户 `KafkaPrincipal.ANONYMOUS`
* `SslAuthenticator`: `authenticate()` 方法为空，`principal()` 方法可以根据规则映射提取，失败返回匿名用户
* `SaslClientAuthenticator`
* `SaslServerAuthenticator`: kafka server 端的 authenticator，`authenticate()` 方法完成对客户端的 SASL 握手，`principal()` 方法返回 SASL 用户

### SaslServerAuthenticator

每个连接关联一种特定的 mechanism，不同 mechanism 对应不同的 callbackHandlers

#### AuthenticateCallbackHandler

* PlainServerCallbackHandler
* ScramServerCallbackHandler
* SaslClientCallbackHandler
* SaslServerCallbackHandler

## Selector

可以监听多个连接，每个连接由 id 以及对应的 KafkaChannel 标识

``` java
private final Map<String, KafkaChannel> channels;
private final Map<String, KChannel> closingKChannels;

private final List<String> connected;
private final List<String> disconnected;
private final Set<SelectionKey> immediatelyConnectedKeys;

private final List<KSend> completedSends;
private final LinkedHashMap<String, KReceive> completedReceives;
```

### connect

``` java
connect(String id, InetSocketAddress address)
```

1. 连接 address，得到 socketChannel
2. 注册 socketChannel 监听 connect 事件
3. 将 socketChannel 包装为 KafkaChannel
4. 将 id 与远程连接的 KafkaChannel 对应起来

### send

``` java
send(KafkaSend send)
```

1. 发送 NetworkSend
    1. 根据 NetworkSend 的 destination 取到对应的 KafkaChannel
    2. 设置 KafkaChannel 的 send

### poll

nio select key，根据 key 找到对应的 KafkaChannel，执行 KafkaChannel 的 read, write 方法

### completedReceives

```
Collection<KReceive> completedReceives()
```

获取完成的响应，每次 poll() 结束后要获取一次，否则下次 poll() 会清空。

### registerChannel

``` java
SelectionKey registerChannel(String id, SocketChannel socketChannel, int interestedOps)
```

1. 注册 socketChannel 监听 interestedOps 事件
2. 将 socketChannel 包装为 KafkaChannel
3. 将 id 与远程连接的 KafkaChannel 对应起来

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

