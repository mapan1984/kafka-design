# Send

``` java
package org.apache.kafka.common.network;

import java.io.IOException;

/**
 * This interface models the in-progress sending of data.
 */
public interface Send {

    /**
     * Is this send complete?
     */
    boolean completed();

    /**
     * Write some as-yet unwritten bytes from this send to the provided channel. It may take multiple calls for the send
     * to be completely written
     * @param channel The Channel to write to
     * @return The number of bytes written
     * @throws IOException If the write fails
     */
    long writeTo(TransferableChannel channel) throws IOException;

    /**
     * Size of the send
     */
    long size();

}
```
