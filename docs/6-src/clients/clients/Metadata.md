# Metadata

``` java
implements Closeable
```

被 the client thread (for partitioning) 和 the background sender thread 共享

Metadata is maintained for only a subset of topics, which can be added to over time. 当我们请求 metadata 中没有的 topic 时会触发 matadata 更新。

如果 topic expiry 被设置，任何在 expiry interval 时间内没有使用的 topic 会被从 metadata 中移除。
消费组禁止 topic expiry 因为它 explicityly manage topics，producers 依赖 topic expiry to limit the refresh set.

## 属性

``` java
// metadata 更新失败时,为避免频繁更新 meta,最小的间隔时间,默认 100ms
private final long refreshBackoffMs;
// metadata 的过期时间, 默认 60,000ms
private final long metadataExpireMs;

private int updateVersion;  // bumped on every metadata response
private int requestVersion; // bumped on every new topic addition


// 最后一次更新的时间（包含更新失败的情况）
private long lastRefreshMs;
// 最后一次成功更新的时间
private long lastSuccessfulRefreshMs;

private KafkaException fatalException;

private Set<String> invalidTopics;
private Set<String> unauthorizedTopics;

private boolean needFullUpdate;
private boolean needPartialUpdate;

private final ClusterResourceListeners clusterResourceListeners;
private boolean isClosed;
private final Map<TopicPartition, Integer> lastSeenLeaderEpochs;


private MetadataCache cache = MetadataCache.empty();
```

### MetadataCache

`MetadataCache`

``` java
private final String clusterId;
private final Map<Integer, Node> nodes;
private final Set<String> unauthorizedTopics;
private final Set<String> invalidTopics;
private final Set<String> internalTopics;
private final Node controller;
private final Map<TopicPartition, PartitionMetadata> metadataByPartition;

private Cluster clusterInstance;
```

## 方法

### add

``` java
```
