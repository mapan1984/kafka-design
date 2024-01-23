# KafkaChannel

channel 表示一个具体的 TCP 连接

属性：

* `id`: 表示 channel 对应的节点。selector 可以根据 id 取到对应的 KafkaChannel

* `Send(NetworkSend)`: 只能持有一个 send，write 发送当前 send 之后，会将 send 置为 null，之后可以设置下一个 send
* `NetworkReceive`: 只能持有一个 receive，read 接收完整数据后，将 receive 置为 null, 之后可以接收下一个 receive

* `TransportLayer`: send, receive 都通过 TransportLayer 转发给 SocketChannel
* `Authenticator`: 提供对连接的认证支持，向 TransportLayer 转发读写

## 用户信息

这个 TCP 关联的用户信息

``` java
    /**
     * Returns the principal returned by `authenticator.principal()`.
     */
    public KafkaPrincipal principal() {
        return authenticator.principal();
    }
```

## prepare

`prepare()` 会先进行 ssl 握手，完成后进行用户认证，在 ready 之前，prepare 会运行多次。

``` java
    public void prepare() throws AuthenticationException, IOException {
        boolean authenticating = false;
        try {
            if (!transportLayer.ready()) {
                transportLayer.handshake();
            }
            if (transportLayer.ready() && !authenticator.complete()) {
                authenticating = true;
                authenticator.authenticate();
            }
        } catch (AuthenticationException e) {
            // Clients are notified of authentication exceptions to enable operations to be terminated
            // without retries. Other errors are handled as network exceptions in Selector.
            String remoteDesc = remoteAddress != null ? remoteAddress.toString() : null;
            state = new ChannelState(ChannelState.State.AUTHENTICATION_FAILED, e, remoteDesc);
            if (authenticating) {
                delayCloseOnAuthenticationFailure();
                throw new DelayedResponseAuthenticationException(e);
            }
            throw e;
        }
        if (ready()) {
            ++successfulAuthentications;
            state = ChannelState.READY;
        }
    }
```

## ready

表示握手与用户认证都已完成

``` java
    public boolean ready() {
        return transportLayer.ready() && authenticator.complete();
    }
```

## 实际读写

channel 的实际读写在 `Selector` 的 `pollSelectionKeys()` 方法中进行。

``` java
    void pollSelectionKeys(Set<SelectionKey> selectionKeys,
                           boolean isImmediatelyConnected,
                           long currentTimeNanos) {
        for (SelectionKey key : determineHandlingOrder(selectionKeys)) {
            KafkaChannel channel = channel(key);

            try {
                /* if channel is not ready finish prepare */
                if (channel.isConnected() && !channel.ready()) {  // channel 未准备好
                    channel.prepare();  // 握手和身份认证，如果执行后还没有 ready，等待下一次 poll 继续执行
                    if (channel.ready()) {
                    }
                }

                if (channel.ready() && channel.state() == ChannelState.NOT_CONNECTED)
                    channel.state(ChannelState.READY);

                if (channel.ready() && (key.isReadable() || channel.hasBytesBuffered()) && !hasCompletedReceive(channel)
                        && !explicitlyMutedChannels.contains(channel)) {
                    attemptRead(channel);
                }

                try {
                    attemptWrite(key, channel, nowNanos);
                } catch (Exception e) {
                }

                /* cancel any defunct sockets */
                if (!key.isValid())
                    close(channel, CloseMode.GRACEFUL);

            } catch (Exception e) {
            } finally {
            }
        }
    }
```

