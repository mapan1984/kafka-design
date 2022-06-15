# Selector

a nioSelector interface for doing non-blocking multi-connection network I/O.

This class works with `NetworkSend` and `NetworkReceive` to transmit size-delimited network requests and responses.

a connection can be added to the nioSelector associated with an integer id by doing.

``` java
nioSelector.connect("42", new InetSocketAddress("google.com", server.port), 64000, 64000);
```

the `connect` call does not block on the creation of the TCP connection, so the `connect` method only begins initiating the connection. the successful invocation of this method does not mean a valid connection has been established.

**Sending requests**, **receiving responses**, **processing connection completions**, and **disconnections on the existing connections** are all done using the `poll()` call.

``` java
nioSelector.send(new NetworkSend(myDestination, myBytes));
nioSelector.send(new NetworkSend(myOtherDestination, myOtherBytes));
nioSelector.poll(TIMEOUT_MS);
```

The nioSelector maintains several lists that are reset by each call to <code>poll()</code> which are available via
various getters. These are reset by each call to <code>poll()</code>.

This class is not thread safe!

## 属性

``` java
private final Logger log;

// java nio Selector
private final java.nio.channels.Selector nioSelector;

// key 为 connection id，值为对应的 KafkaChannel
private final Map<String, KafkaChannel> channels;
// 
private final Set<KafkaChannel> explicitlyMutedChannels;

private boolean outOfMemory;

// 调用 poll 之后，记录已发送完成的 Send
private final List<NetworkSend> completedSends;
// 调用 poll 之后，记录已接收完成的 Receive
private final LinkedHashMap<String, NetworkReceive> completedReceives;
// 调用 poll 之后，记录已连接 connection 对应的 id
private final List<String> connected;
// 调用 poll 之后，记录已断开连接 connection 对应的 id 和 状态
private final Map<String, ChannelState> disconnected;
// 调用 poll 之后，记录发送失败的 Send 对应的 connection id
private final List<String> failedSends;

// 记录立即已连接的 connection key
private final Set<SelectionKey> immediatelyConnectedKeys;

// 
private final Map<String, KafkaChannel> closingChannels;

// 
private Set<SelectionKey> keysWithBufferedRead;



private final Time time;
private final SelectorMetrics sensors;
private final ChannelBuilder channelBuilder;
// Max size in bytes of a single network receive (use {@link NetworkReceive#UNLIMITED} for no limit)jk:w
private final int maxReceiveSize;
private final boolean recordTimePerConnection;
private final IdleExpiryManager idleExpiryManager;
private final LinkedHashMap<String, DelayedAuthenticationFailureClose> delayedClosingChannels;
private final MemoryPool memoryPool;
private final long lowMemThreshold;
private final int failedAuthenticationDelayMs;
```

## 方法

构造方法中，执行：

``` java
try {
    this.nioSelector = java.nio.channels.Selector.open();
} catch (IOException e) {
    throw new KafkaException(e);
}
```

### connect

``` java
/**
 * Begin connecting to the given address and add the connection to this nioSelector associated with the given id
 * number.
 * <p>
 * Note that this call only initiates the connection, which will be completed on a future {@link #poll(long)}
 * call. Check {@link #connected()} to see which (if any) connections have completed after a given poll call.
 * @param id The id for the new connection
 * @param address The address to connect to
 * @param sendBufferSize The send buffer for the new connection
 * @param receiveBufferSize The receive buffer for the new connection
 * @throws IllegalStateException if there is already a connection for that id
 * @throws IOException if DNS resolution fails on the hostname or if the broker is down
 */
@Override
public void connect(String id, InetSocketAddress address, int sendBufferSize, int receiveBufferSize) throws IOException {
    ensureNotRegistered(id);
    SocketChannel socketChannel = SocketChannel.open();
    SelectionKey key = null;
    try {
        configureSocketChannel(socketChannel, sendBufferSize, receiveBufferSize);
        // 调用 channel.connect(address)
        boolean connected = doConnect(socketChannel, address);
        // 1. 将 channel 注册到 nioSelector，获得 SelectionKey
        // 2. 构造 KafkaChannel
        // 3. 将 id, KafkaChannel 对应关系记录到 this.channels
        // 4. 如果有 idelExpiryManage，将 connection id 加入
        // 5. 并返回 SelectionKey
        key = registerChannel(id, socketChannel, SelectionKey.OP_CONNECT);

        if (connected) {
            // OP_CONNECT won't trigger for immediately connected channels
            log.debug("Immediately connected to node {}", id);
            immediatelyConnectedKeys.add(key);
            key.interestOps(0);
        }
    } catch (IOException | RuntimeException e) {
        if (key != null)
            immediatelyConnectedKeys.remove(key);
        channels.remove(id);
        socketChannel.close();
        throw e;
    }
}
```

### send

``` java
/**
 * Queue the given request for sending in the subsequent {@link #poll(long)} calls
 * @param send The request to send
 */
public void send(NetworkSend send) {
    // 1. destinationId 即为 this.channels 中的 connectionId
    String connectionId = send.destinationId();
    // 2. 根据 id 从 this.channels 或者 closingChannels 中获取 id 对应的 KafkaChannel
    KafkaChannel channel = openOrClosingChannelOrFail(connectionId);
    if (closingChannels.containsKey(connectionId)) {
        // ensure notification via `disconnected`, leave channel in the state in which closing was triggered
        this.failedSends.add(connectionId);
    } else {
        try {
            // 3.1. 将 KafkaChannel 的 send 设置为 send
            // 3.2. TransportLayer SelectionKey 设置 OP_WRITE 事件
            channel.setSend(send);
        } catch (Exception e) {
            // update the state for consistency, the channel will be discarded after `close`
            channel.state(ChannelState.FAILED_SEND);
            // ensure notification via `disconnected` when `failedSends` are processed in the next poll
            this.failedSends.add(connectionId);
            close(channel, CloseMode.DISCARD_NO_NOTIFY);
            if (!(e instanceof CancelledKeyException)) {
                log.error("Unexpected exception during send, closing connection {} and rethrowing exception {}",
                        connectionId, e);
                throw e;
            }
        }
    }
}
```

### poll

``` java
/**
 * Do whatever I/O can be done on each connection without blocking. This includes completing connections, completing
 * disconnections, initiating new sends, or making progress on in-progress sends or receives.
 *
 * When this call is completed the user can check for completed sends, receives, connections or disconnects using
 * {@link #completedSends()}, {@link #completedReceives()}, {@link #connected()}, {@link #disconnected()}. These
 * lists will be cleared at the beginning of each `poll` call and repopulated by the call if there is
 * any completed I/O.
 *
 * In the "Plaintext" setting, we are using socketChannel to read & write to the network. But for the "SSL" setting,
 * we encrypt the data before we use socketChannel to write data to the network, and decrypt before we return the responses.
 * This requires additional buffers to be maintained as we are reading from network, since the data on the wire is encrypted
 * we won't be able to read exact no.of bytes as kafka protocol requires. We read as many bytes as we can, up to SSLEngine's
 * application buffer size. This means we might be reading additional bytes than the requested size.
 * If there is no further data to read from socketChannel selector won't invoke that channel and we have additional bytes
 * in the buffer. To overcome this issue we added "keysWithBufferedRead" map which tracks channels which have data in the SSL
 * buffers. If there are channels with buffered data that can by processed, we set "timeout" to 0 and process the data even
 * if there is no more data to read from the socket.
 *
 * Atmost one entry is added to "completedReceives" for a channel in each poll. This is necessary to guarantee that
 * requests from a channel are processed on the broker in the order they are sent. Since outstanding requests added
 * by SocketServer to the request queue may be processed by different request handler threads, requests on each
 * channel must be processed one-at-a-time to guarantee ordering.
 *
 * @param timeout The amount of time to wait, in milliseconds, which must be non-negative
 * @throws IllegalArgumentException If `timeout` is negative
 * @throws IllegalStateException If a send is given for which we have no existing connection or for which there is
 *         already an in-progress send
 */
@Override
public void poll(long timeout) throws IOException {
    if (timeout < 0)
        throw new IllegalArgumentException("timeout should be >= 0");

    boolean madeReadProgressLastCall = madeReadProgressLastPoll;
    // 1. 清空 completedSends, completedReceives 等记录
    clear();

    boolean dataInBuffers = !keysWithBufferedRead.isEmpty();

    if (!immediatelyConnectedKeys.isEmpty() || (madeReadProgressLastCall && dataInBuffers))
        timeout = 0;

    if (!memoryPool.isOutOfMemory() && outOfMemory) {
        //we have recovered from memory pressure. unmute any channel not explicitly muted for other reasons
        log.trace("Broker no longer low on memory - unmuting incoming sockets");
        for (KafkaChannel channel : channels.values()) {
            if (channel.isInMutableState() && !explicitlyMutedChannels.contains(channel)) {
                channel.maybeUnmute();
            }
        }
        outOfMemory = false;
    }

    /* check ready keys */
    long startSelect = time.nanoseconds();
    // 调用 java nio select，并返回（select 这里会阻塞）
    int numReadyKeys = select(timeout);
    long endSelect = time.nanoseconds();
    this.sensors.selectTime.record(endSelect - startSelect, time.milliseconds());

    if (numReadyKeys > 0 || !immediatelyConnectedKeys.isEmpty() || dataInBuffers) {
        // 获取有事件发生的 key
        Set<SelectionKey> readyKeys = this.nioSelector.selectedKeys();

        // Poll from channels that have buffered data (but nothing more from the underlying socket)
        if (dataInBuffers) {
            keysWithBufferedRead.removeAll(readyKeys); //so no channel gets polled twice
            Set<SelectionKey> toPoll = keysWithBufferedRead;
            keysWithBufferedRead = new HashSet<>(); //poll() calls will repopulate if needed
            pollSelectionKeys(toPoll, false, endSelect);
        }

        // Poll from channels where the underlying socket has more data
        pollSelectionKeys(readyKeys, false, endSelect);
        // Clear all selected keys so that they are included in the ready count for the next select
        readyKeys.clear();

        pollSelectionKeys(immediatelyConnectedKeys, true, endSelect);
        immediatelyConnectedKeys.clear();
    } else {
        madeReadProgressLastPoll = true; //no work is also "progress"
    }

    long endIo = time.nanoseconds();
    this.sensors.ioTime.record(endIo - endSelect, time.milliseconds());

    // Close channels that were delayed and are now ready to be closed
    completeDelayedChannelClose(endIo);

    // we use the time at the end of select to ensure that we don't close any connections that
    // have just been processed in pollSelectionKeys
    maybeCloseOldestConnection(endSelect);
}
```

### pollSelectionKeys

``` java
    /**
     * handle any ready I/O on a set of selection keys
     * @param selectionKeys set of keys to handle
     * @param isImmediatelyConnected true if running over a set of keys for just-connected sockets
     * @param currentTimeNanos time at which set of keys was determined
     */
    // package-private for testing
    void pollSelectionKeys(Set<SelectionKey> selectionKeys,
                           boolean isImmediatelyConnected,
                           long currentTimeNanos) {
        for (SelectionKey key : determineHandlingOrder(selectionKeys)) {
            KafkaChannel channel = channel(key);
            long channelStartTimeNanos = recordTimePerConnection ? time.nanoseconds() : 0;
            boolean sendFailed = false;
            String nodeId = channel.id();

            // register all per-connection metrics at once
            sensors.maybeRegisterConnectionMetrics(nodeId);
            if (idleExpiryManager != null)
                idleExpiryManager.update(nodeId, currentTimeNanos);

            try {
                /* complete any connections that have finished their handshake (either normally or immediately) */
                if (isImmediatelyConnected || key.isConnectable()) {
                    if (channel.finishConnect()) {
                        this.connected.add(nodeId);
                        this.sensors.connectionCreated.record();

                        SocketChannel socketChannel = (SocketChannel) key.channel();
                        log.debug("Created socket with SO_RCVBUF = {}, SO_SNDBUF = {}, SO_TIMEOUT = {} to node {}",
                                socketChannel.socket().getReceiveBufferSize(),
                                socketChannel.socket().getSendBufferSize(),
                                socketChannel.socket().getSoTimeout(),
                                nodeId);
                    } else {
                        continue;
                    }
                }

                /* if channel is not ready finish prepare */
                if (channel.isConnected() && !channel.ready()) {
                    channel.prepare();
                    if (channel.ready()) {
                        long readyTimeMs = time.milliseconds();
                        boolean isReauthentication = channel.successfulAuthentications() > 1;
                        if (isReauthentication) {
                            sensors.successfulReauthentication.record(1.0, readyTimeMs);
                            if (channel.reauthenticationLatencyMs() == null)
                                log.warn(
                                    "Should never happen: re-authentication latency for a re-authenticated channel was null; continuing...");
                            else
                                sensors.reauthenticationLatency
                                    .record(channel.reauthenticationLatencyMs().doubleValue(), readyTimeMs);
                        } else {
                            sensors.successfulAuthentication.record(1.0, readyTimeMs);
                            if (!channel.connectedClientSupportsReauthentication())
                                sensors.successfulAuthenticationNoReauth.record(1.0, readyTimeMs);
                        }
                        log.debug("Successfully {}authenticated with {}", isReauthentication ?
                            "re-" : "", channel.socketDescription());
                    }
                }
                if (channel.ready() && channel.state() == ChannelState.NOT_CONNECTED)
                    channel.state(ChannelState.READY);
                Optional<NetworkReceive> responseReceivedDuringReauthentication = channel.pollResponseReceivedDuringReauthentication();
                responseReceivedDuringReauthentication.ifPresent(receive -> {
                    long currentTimeMs = time.milliseconds();
                    addToCompletedReceives(channel, receive, currentTimeMs);
                });

                //if channel is ready and has bytes to read from socket or buffer, and has no
                //previous completed receive then read from it
                if (channel.ready() && (key.isReadable() || channel.hasBytesBuffered()) && !hasCompletedReceive(channel)
                        && !explicitlyMutedChannels.contains(channel)) {
                    attemptRead(channel);
                }

                if (channel.hasBytesBuffered()) {
                    //this channel has bytes enqueued in intermediary buffers that we could not read
                    //(possibly because no memory). it may be the case that the underlying socket will
                    //not come up in the next poll() and so we need to remember this channel for the
                    //next poll call otherwise data may be stuck in said buffers forever. If we attempt
                    //to process buffered data and no progress is made, the channel buffered status is
                    //cleared to avoid the overhead of checking every time.
                    keysWithBufferedRead.add(key);
                }

                /* if channel is ready write to any sockets that have space in their buffer and for which we have data */

                long nowNanos = channelStartTimeNanos != 0 ? channelStartTimeNanos : currentTimeNanos;
                try {
                    attemptWrite(key, channel, nowNanos);
                } catch (Exception e) {
                    sendFailed = true;
                    throw e;
                }

                /* cancel any defunct sockets */
                if (!key.isValid())
                    close(channel, CloseMode.GRACEFUL);

            } catch (Exception e) {
                String desc = channel.socketDescription();
                if (e instanceof IOException) {
                    log.debug("Connection with {} disconnected", desc, e);
                } else if (e instanceof AuthenticationException) {
                    boolean isReauthentication = channel.successfulAuthentications() > 0;
                    if (isReauthentication)
                        sensors.failedReauthentication.record();
                    else
                        sensors.failedAuthentication.record();
                    String exceptionMessage = e.getMessage();
                    if (e instanceof DelayedResponseAuthenticationException)
                        exceptionMessage = e.getCause().getMessage();
                    log.info("Failed {}authentication with {} ({})", isReauthentication ? "re-" : "",
                        desc, exceptionMessage);
                } else {
                    log.warn("Unexpected error from {}; closing connection", desc, e);
                }

                if (e instanceof DelayedResponseAuthenticationException)
                    maybeDelayCloseOnAuthenticationFailure(channel);
                else
                    close(channel, sendFailed ? CloseMode.NOTIFY_ONLY : CloseMode.GRACEFUL);
            } finally {
                maybeRecordTimePerConnection(channel, channelStartTimeNanos);
            }
        }
    }
```

### wakeup

``` java
/**
 * Interrupt the nioSelector if it is blocked waiting to do I/O.
 */
@Override
public void wakeup() {
    this.nioSelector.wakeup();
}
```

### write

``` java
// package-private for testing
void write(KafkaChannel channel) throws IOException {
    String nodeId = channel.id();
    long bytesSent = channel.write();
    Send send = channel.maybeCompleteSend();
    // We may complete the send with bytesSent < 1 if `TransportLayer.hasPendingWrites` was true and `channel.write()`
    // caused the pending writes to be written to the socket channel buffer
    if (bytesSent > 0 || send != null) {
        long currentTimeMs = time.milliseconds();
        if (bytesSent > 0)
            this.sensors.recordBytesSent(nodeId, bytesSent, currentTimeMs);
        if (send != null) {
            this.completedSends.add(send);
            this.sensors.recordCompletedSend(nodeId, send.size(), currentTimeMs);
        }
    }
}
```
