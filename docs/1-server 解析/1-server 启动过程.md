# 1.1 server 启动过程

## 启动

在启动脚本 `kafka-server-start.sh` 中可以看到，运行类为 `kafka.Kafka`，其 `main` 函数即为服务启动的入口。

查看 `Kafka` 的 `main` 函数，其内容为：

1. 解析参数，第一个参数为配置文件路径，从配置文件加载服务配置。其他参数可选，指定服务配置。
2. 构建 `KafkaServer` (2.8.0 之后支持 `KafkaRaftServer`)
3. 对构造的 `KafkaServer` 对象调用 `startup()` 方法

查看 [KafkaServer](/kafka-design/6-src/core/kafka/server/KafkaServer) 代码，其 `startup` 执行以下步骤：

1. initZkClient
    1. createZkClient
2. get or create cluster_id
3. load metadata
4. check cluster id
5. generate brokerId
6. initialize dynamic borker configs from ZooKeeper
7. start scheduler
8. create and configure metrics

