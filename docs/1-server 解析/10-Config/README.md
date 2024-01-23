# 配置

Kafka 服务启动时，在 `main()` 方法中通过 `getPropsFromArgs()` 获取命令行参数，并读取指定的 `server.properties` 文件，获取服务的配置参数。

``` scala
  def main(args: Array[String]): Unit = {
    try {
      val serverProps = getPropsFromArgs(args)
      val kafkaServerStartable = KafkaServerStartable.fromProps(serverProps)
      // ...
    }
    catch {
    }
  }
```

``` scala
  def getPropsFromArgs(args: Array[String]): Properties = {
    if (args.length == 0 || args.contains("--help")) {
      CommandLineUtils.printUsageAndDie(optionParser, "USAGE: java [options] %s server.properties [--override property=value]*".format(classOf[KafkaServer].getSimpleName()))
    }

    if (args.contains("--version")) {
      CommandLineUtils.printVersionAndDie()
    }

    // NOTE: args(0) 就是启动命令指定的 server.properties 文件
    val props = Utils.loadProps(args(0))

    if (args.length > 1) {
    }
    props
  }
```

在 `KafkaServerStartable.fromProps()` 中，会进一步把 props 解析为 `KafkaConfig` 实例

``` scala
object KafkaServerStartable {
  def fromProps(serverProps: Properties): KafkaServerStartable = {
    fromProps(serverProps, None)
  }

  def fromProps(serverProps: Properties, threadNamePrefix: Option[String]): KafkaServerStartable = {
    val reporters = KafkaMetricsReporter.startReporters(new VerifiableProperties(serverProps))
    // NOTE: KafkaConfig.fromProps 方法会根据 serverProps 构建 KafkaConfig 实例
    new KafkaServerStartable(KafkaConfig.fromProps(serverProps, false), reporters, threadNamePrefix)
  }
}
```

