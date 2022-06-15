# Receive

``` java
package org.apache.kafka.common.network;

import java.io.Closeable;
import java.io.IOException;
import java.nio.channels.ScatteringByteChannel;

/**
 * This interface models the in-progress reading of data from a channel to a source identified by an integer id
 */
public interface Receive extends Closeable {

    /**
     * The numeric id of the source from which we are receiving data.
     */
    String source();

    /**
     * Are we done receiving data?
     */
    boolean complete();

    /**
     * Read bytes into this receive from the given channel
     * @param channel The channel to read from
     * @return The number of bytes read
     * @throws IOException If the reading fails
     */
    long readFrom(ScatteringByteChannel channel) throws IOException;

    /**
     * Do we know yet how much memory we require to fully read this
     */
    boolean requiredMemoryAmountKnown();

    /**
     * Has the underlying memory required to complete reading been allocated yet?
     */
    boolean memoryAllocated();
}
```
