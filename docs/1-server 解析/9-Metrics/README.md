# Metrics

`Metrics` 类管理所有 metrics 和 sensor

KafkaServer startup() 方法中会初始化一个 `Metrics` 实例

管理成员：


``` java
// 所有注册的 metric，通过 addMetric() 方法添加
private final ConcurrentMap<MetricName, KafkaMetric> metrics;

// 所有注册的 Sensor，通过 sensor() 方法添加
private final ConcurrentMap<String, Sensor> sensors;
// key 为 parent sensor，value 为这个 parent 拥有的 children sensor
private final ConcurrentMap<Sensor, List<Sensor>> childrenSensors;

private final List<MetricsReporter> reporters;
```

## MetricName

监控标识：

- name
- group
- description
- tags

## KafkaMetric

监控项，

包含一个 `MetricConfig` 对象，`MetricConfig` 对象记录:

``` scala
// 限流机制
private Quota quota;

// 样本数
private int samples;

// 单个样本的事件窗口大小
private long eventWindow;

// 单个样本的时间窗口大小
private long timeWindowMs;

private Map<String, String> tags;
private Sensor.RecordingLevel recordingLevel;
```

包含一个 `MetricValueProvider` 对象，用来提供监控值。

`MetricValueProvider` 是一个空的 `interface`，其有 2 个继承 `interface`

- `Measurable`
- `Gauge`

### SampledStat

`SampledStat` 实现了 `Measurable` 接口，其以及其继承类可以作为 `KafkaMetric` 的 `MetricValueProvider`

`SampledStat` 中记录：

``` java
// 样本初始值
private double initialValue;

// 当前样本下标索引
private int current = 0;

// 样本列表
protected List<Sample> samples;
```

`Sample` 中记录：

``` java
// 初始值
public double initialValue;

// 样本事件数
public long eventCount;

// 窗口起始时间
public long lastWindowMs;

// 窗口值
public double value;

```

`SampledStat` 通过 `record()` 方法记录数据：

``` java
@Override
public void record(MetricConfig config, double value, long timeMs) {
    // 1. 获取当前 sample，如果没有就新建一个
    Sample sample = current(timeMs);

    // 2. 检查当前 sample 窗口是否结束
    if (sample.isComplete(timeMs, config))
        // 2.1 如果当前 sample 窗口结束，current +1，移向下一个 sample
        sample = advance(config, timeMs);

    // 3. 在当前 sample 中记录 value，当前 sample 的 value 记录，
    //    这里不同的统计方法有不同的实现，例如 avg, max, min 等
    update(sample, config, value, timeMs);

    // 4. 当前 sample 的事件数 +1
    sample.eventCount += 1;
}
```

`SampledStat` 通过 `measure()` 方法得到样本的统计值：

``` java
@Override
public double measure(MetricConfig config, long now) {
    // 1. 重置所有过期的 sample
    purgeObsoleteSamples(config, now);

    // 2. 计算所有 sample 的统计值，这里不同的统计方法有不同的实现，例如 avg, max, min 等
    return combine(this.samples, config, now);
}


/* Timeout any windows that have expired in the absence of any events */
protected void purgeObsoleteSamples(MetricConfig config, long now) {
    long expireAge = config.samples() * config.timeWindowMs();
    for (Sample sample : samples) {
        if (now - sample.lastWindowMs >= expireAge)
            sample.reset(now);
    }
}
```

#### Avg

继承自 `SampledStat`，复写 `update()`, `combine()`

``` java
@Override
protected void update(Sample sample, MetricConfig config, double value, long now) {
    sample.value += value;
}

@Override
public double combine(List<Sample> samples, MetricConfig config, long now) {
    double total = 0.0;
    long count = 0;
    for (Sample s : samples) {
        total += s.value;
        count += s.eventCount;
    }
    return count == 0 ? Double.NaN : total / count;
}
```

#### Max

继承自 `SampledStat`，复写 `update()`, `combine()`

``` java
@Override
protected void update(Sample sample, MetricConfig config, double value, long now) {
    sample.value = Math.max(sample.value, value);
}

@Override
public double combine(List<Sample> samples, MetricConfig config, long now) {
    double max = Double.NEGATIVE_INFINITY;
    long count = 0;
    for (Sample sample : samples) {
        max = Math.max(max, sample.value);
        count += sample.eventCount;
    }
    return count == 0 ? Double.NaN : max;
}
```

### Rate

`Rate` 实现了 `Measurable` 接口，其以及其继承类可以作为 `KafkaMetric` 的 `MetricValueProvider`

``` java
protected final TimeUnit unit;
protected final SampledStat stat;
```

``` java
@Override
public void record(MetricConfig config, double value, long timeMs) {
    // 记录数据通过持有的 sampledStat 完成
    this.stat.record(config, value, timeMs);
}

@Override
public double measure(MetricConfig config, long now) {
    // 获得统计值，先由持有的 sampledStat 的到样本统计值，在除以时间得到速率
    double value = stat.measure(config, now);
    return value / convert(windowSize(config, now), unit);
}
```

## Sensor

通过 `Stat` 记录监控数据值

``` java
   private void recordInternal(double value, long timeMs, boolean checkQuotas) {
        this.lastRecordTime = timeMs;
        synchronized (this) {
            synchronized (metricLock()) {
                // increment all the stats
                for (StatAndConfig statAndConfig : this.stats) {
                    statAndConfig.stat.record(statAndConfig.config(), value, timeMs);
                }
            }
            if (checkQuotas)
                checkQuotas(timeMs);
        }
        for (Sensor parent : parents)
            parent.record(value, timeMs, checkQuotas);
    }
```

`Stat` 有多种实现，例如在记录数据时完成 max, min, avg 等数据的更新

``` java
/**
 * A Stat is a quantity such as average, max, etc that is computed off the stream of updates to a sensor
 */
public interface Stat {

    /**
     * Record the given value
     * @param config The configuration to use for this metric
     * @param value The value to record
     * @param timeMs The POSIX time in milliseconds this value occurred
     */
    void record(MetricConfig config, double value, long timeMs);

}
```
