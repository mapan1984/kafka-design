# NetworkSend

``` java
public class NetworkSend implements Send {
    private final String destinationId;
    private final Send send;

    public NetworkSend(String destinationId, Send send) {
        this.destinationId = destinationId;
        this.send = send;
    }

    public String destinationId() {
        return destinationId;
    }

    @Override
    public boolean completed() {
        return send.completed();
    }

    @Override
    public long writeTo(TransferableChannel channel) throws IOException {
        return send.writeTo(channel);
    }

    @Override
    public long size() {
        return send.size();
    }

}
```
