# Kafka Admin Client

## 0.10.0.0 ~ 0.10.2.2

只有 `AdminClient.scala`，提供了通用的发送请求方法：

* send
* sendAnyNode

和一些 broker, consumer 相关的方法：

* findAllBrokers
* findCoordinator
* listGroups
* listAllGroups
* listAllConsumerGroups
* listAllGroupsFlattened
* listAllConsumerGroupsFlattened
* describeGroup
* describeConsumerGroup

* listGroupOffsets
* listAllBrokerVersionInfo

## 0.11.0.0 ~ 0.11.0.2

增加抽象类 `AdminClient.java` 和其实现类 `KafkaAdminClient.java`，`AdminClient.scala` 被标记为 deprected。

提供了以下方法：

* createTopics
    0.10.1.0 或者更高版本的 broker 可用，validateOnly 选项需要 0.10.2.0 及以上版本
* deleteTopics
    0.10.1.0 或者更高版本的 broker 可用
* listTopics
* describeTopics

* describeCluster

* describeAcls
    0.11.0.0 或者更高版本的 broker 可用
* createAcls
    0.11.0.0 或者更高版本的 broker 可用
* deleteAcls
    0.11.0.0 或者更高版本的 broker 可用

* describeConfigs
    0.11.0.0 或者更高版本的 broker 可用
* alterConfigs
    0.11.0.0 或者更高版本的 broker 可用
    2.3.0 deprecated

## 1.0.0 ~ 1.0.2

* alterReplicaLogDirs
    1.0.0 或者更高版本的 broker 可用
    2.2.0 要求 1.1.0 或更高版本 broker 可用
* describeLogDirs
    1.0.0 或者更高版本的 broker 可用
* describeReplicaLogDirs
    1.0.0 或者更高版本的 broker 可用
* createPartitions
    1.0.0 或者更高版本的 broker 可用

## 1.1.0 ~ 1.1.1

* deleteRecords
    0.11.0.0 或者更高版本的 broker 可用

## 2.0.0 ~ 2.1.1

* createDelegationToken
    1.1.0 或者更高版本的 broker 可用
* renewDelegationToken
    1.1.0 或者更高版本的 broker 可用
* expireDelegationToken
    1.1.0 或者更高版本的 broker 可用
* describeDelegationToken
    1.1.0 或者更高版本的 broker 可用
* describeConsumerGroups
* listConsumerGroups
* listConsumerGroupOffsets
* deleteConsumerGroups

## 2.2.0 ~ 2.2.2

* electPreferredLeaders
    2.2.0 或更高版本broker
    deprecated·Since·2.4.0
    3.0.0 删除

## 2.3.0 ~ 2.3.1

* incrementalAlterConfigs
    要求 2.3.0 或更高版本 broker

## 2.4.0 ~ 2.4.1

增加了接口 `Admin.java`，抽象类 `AdminClient.java` 被标记为 deprecated

* deleteConsumerGroupOffsets
* electLeaders
    最优 leader 选举需要 2.2.0 或更高 broker，其他类型需要 2.4.0 或更高 broker
* alterPartitionReassignments
* listPartitionReassignments
* removeMembersFromConsumerGroup

## 2.5.0 ~ 2.5.1

* alterConsumerGroupOffsets
* listOffsets

## 2.6.0 ~ 2.6.3

* describeClientQuotas
    要求 2.6.0 或更高版本 broker
* alterClientQuotas
    要求 2.6.0 或更高版本 broker

## 2.7.0 ~ 2.7.2

* describeUserScramCredentials
    要求 2.7.0 或更高版本 broker
* alterUserScramCredentials
    要求 2.7.0 或更高版本 broker
* describeFeatures
* updateFeatures
    要求 2.7.0 或更高版本 broker

## 2.8.0 ~ 2.8.2

* unregisterBroker
    适用于 kraft 集群

## 3.0.0 ~ 3.0.2

* deleteTopics
    当使用 topic id 时要求 2.8 或者更高
    当使用 topic name 时要求 0.10.1.0 或者更高
* describeProducers
* describeTransactions
* abortTransaction
* listTransactions

## 3.1.0 ~ 3.1.2

* describeTopics
    当使用 topic id 时要求 3.1.0 或者更高

## 3.2.0 ~ 3.2.3

* fenceProducers

## 3.3.0 ~

* listConsumerGroupOffsets
    可以指定具体的 topic partition
* describeMetadataQuorum

