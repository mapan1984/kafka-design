# 5 release note

感兴趣的 KIP

## 重分区 API

增加了重分区区的 API，相比基于 zookeeper 的方式，这种重分区可以终止，可以在上一个还在执行时增加任务
https://cwiki.apache.org/confluence/display/KAFKA/KIP-455%3A+Create+an+Administrative+API+for+Replica+Reassignment

## kafka consumer 支持从 replica 拉数据

https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica

## controller 支持 AlterIsrRequest

可以不再依赖 zookeeper `/isr_change_notification` 传递 isr 变化信息

https://cwiki.apache.org/confluence/display/KAFKA/KIP-497%3A+Add+inter-broker+API+to+alter+ISR

## 消费组在单独的线程发送心跳

https://cwiki.apache.org/confluence/display/KAFKA/KIP-62%3A+Allow+consumer+to+send+heartbeats+from+a+background+thread
