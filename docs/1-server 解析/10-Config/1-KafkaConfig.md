# KafkaConfig

## 配置定义

### KafkaConfig Object

`KafkaConfig` object 通过 `configDef` 定义了所有配置参数的名称、类型、默认值等属性。

``` java
  private val configDef = {
    import ConfigDef.Importance._
    import ConfigDef.Range._
    import ConfigDef.Type._
    import ConfigDef.ValidString._

    new ConfigDef()

      /** ********* Zookeeper Configuration ***********/
      .define(ZkConnectProp, STRING, HIGH, ZkConnectDoc)
      .define(ZkSessionTimeoutMsProp, INT, Defaults.ZkSessionTimeoutMs, HIGH, ZkSessionTimeoutMsDoc)
      .define(ZkConnectionTimeoutMsProp, INT, null, HIGH, ZkConnectionTimeoutMsDoc)
      .define(ZkSyncTimeMsProp, INT, Defaults.ZkSyncTimeMs, LOW, ZkSyncTimeMsDoc)

      ...
      ...

  }
```

### ConfigDef/ConfigKey

`configDef` 是一个 `ConfigDef` 实例，`ConfigDef` 实例维护所有参数定义到 `configKeys` 中，通过 `define()` 方法添加。

``` java
    private final Map<String, ConfigKey> configKeys;
    private final List<String> groups;
    private Set<String> configsWithNoParent;
```

`configKeys` 由 `ConfigKey` 实例组成，可以定义一个配置的名字、类型、默认值、文档等相关属性。

``` java
        public final String name;
        public final Type type;
        public final String documentation;
        public final Object defaultValue;
        public final Validator validator;
        public final Importance importance;
        public final String group;
        public final int orderInGroup;
        public final Width width;
        public final String displayName;
        public final List<String> dependents;
        public final Recommender recommender;
        public final boolean internalConfig;
```

ConfigDef 提供 `parse()` 方法，将 `props` 按照 `configKeys` 的定义解析为对应的 `Map<String, Object>` 类型。

``` java
    public Map<String, Object> parse(Map<?, ?> props) {
        // Check all configurations are defined
        List<String> undefinedConfigKeys = undefinedDependentConfigs();
        if (!undefinedConfigKeys.isEmpty()) {
            String joined = Utils.join(undefinedConfigKeys, ",");
            throw new ConfigException("Some configurations in are referred in the dependents, but not defined: " + joined);
        }
        // parse all known keys
        Map<String, Object> values = new HashMap<>();
        for (ConfigKey key : configKeys.values())
            values.put(key.name, parseValue(key, props.get(key.name), props.containsKey(key.name)));
        return values;
    }
```

## 配置赋值

`KafkaConfig` class 继承自 `AbstractConfig`，

### AbstractConfig

`AbstractConfig` 类有以下属性：


``` java
    /* configs for which values have been requested, used to detect unused configs */
    private final Set<String> used;

    /* the original values passed in by the user */
    // 原始的 props 值
    private final Map<String, ?> originals;

    /* the parsed values */
    // 经过 definition.parse(originals) 后得到的值
    private final Map<String, Object> values;

    // 配置定义：KafkaConfig.configDef
    private final ConfigDef definition;
```

### KafkaConfig Class

`KafkaConfig` class 继承自 `AbstractConfig`，定义了所有配置并赋值。

`KafkaConfig` class 还定义了 `dynamicConfig`，其是 `DynamicBrokerConfig` 实例

``` scala
  private[server] val dynamicConfig = dynamicConfigOverride.getOrElse(new DynamicBrokerConfig(this))

  def addReconfigurable(reconfigurable: Reconfigurable): Unit = {
    dynamicConfig.addReconfigurable(reconfigurable)
  }

  def removeReconfigurable(reconfigurable: Reconfigurable): Unit = {
    dynamicConfig.removeReconfigurable(reconfigurable)
  }
```
