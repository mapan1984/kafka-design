# KafkaChannel

## 属性

``` java
// 唯一的 id 标识
private final String id;
// 读写都会转交个 transportLayer
private final TransportLayer transportLayer;
private final Supplier<Authenticator> authenticatorCreator;
// 用来 authentication （ 或者 re-authentication），读写同样通过 transportLayer 进行
private Authenticator authenticator;
// Tracks accumulated network thread time. This is updated on the network thread.
// The values are read and reset after each response is sent.
private long networkThreadTimeNanos;
private final int maxReceiveSize;
private final MemoryPool memoryPool;
private final ChannelMetadataRegistry metadataRegistry;
// 暂存接收的数据
private NetworkReceive receive;
// 将要发送的数据
private NetworkSend send;
// Track connection and mute state of channels to enable outstanding requests on channels to be
// processed after the channel is disconnected.
private boolean disconnected;
private ChannelMuteState muteState;
private ChannelState state;
private SocketAddress remoteAddress;
private int successfulAuthentications;
private boolean midWrite;
private long lastReauthenticationStartNanos;
```

## 方法

### setSend

``` java
public void setSend(Send send) {
    // 一个 channel 只能暂存一个 Send，当前 Send 未发送前不能发送下一个
    if (this.send != null)
        throw new IllegalStateException("Attempt to begin a send operation with prior send operation still in progress, connection id is " + id);
    this.send = send;
    this.transportLayer.addInterestOps(SelectionKey.OP_WRITE);
}
```

### read & receive

``` java
public long read() throws IOException {
    if (receive == null) {
        receive = new NetworkReceive(maxReceiveSize, id, memoryPool);
    }

    long bytesReceived = receive(this.receive);

    if (this.receive.requiredMemoryAmountKnown() && !this.receive.memoryAllocated() && isInMutableState()) {
        //pool must be out of memory, mute ourselves.
        mute();
    }
    return bytesReceived;
}
```

``` java
private long receive(NetworkReceive receive) throws IOException {
    try {
        return receive.readFrom(transportLayer);
    } catch (SslAuthenticationException e) {
        // With TLSv1.3, post-handshake messages may throw SSLExceptions, which are
        // handled as authentication failures
        String remoteDesc = remoteAddress != null ? remoteAddress.toString() : null;
        state = new ChannelState(ChannelState.State.AUTHENTICATION_FAILED, e, remoteDesc);
        throw e;
    }
}
```

### write

``` java
public long write() throws IOException {
    if (send == null)
        return 0;

    midWrite = true;
    return send.writeTo(transportLayer);
}
```
