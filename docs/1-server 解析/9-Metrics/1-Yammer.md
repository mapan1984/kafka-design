# Yammer metrics

kafka 监控实现依赖 https://github.com/dropwizard/metrics

* Gauges：瞬时值
* Counters：累计值，可以在过去的基础上增加或减少
* Meters：一段时间内的事件发生的速率(Rate)
* Histograms：一个时间内的统计值(Max, Min, Avg)
* Timers：
* Healthy checks

## KafkaYammerMetrics

``` scala
        /* create and configure metrics */
        kafkaYammerMetrics = KafkaYammerMetrics.INSTANCE
        kafkaYammerMetrics.configure(config.originals)

        val jmxReporter = new JmxReporter()
        jmxReporter.configure(config.originals)

        val reporters = new util.ArrayList[MetricsReporter]
        reporters.add(jmxReporter)

        val metricConfig = KafkaServer.metricConfig(config)
        val metricsContext = createKafkaMetricsContext()
        metrics = new Metrics(metricConfig, reporters, time, true, metricsContext)
```

