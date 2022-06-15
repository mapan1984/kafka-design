# 1.7 Acl 权限控制

kafka 提供了权限控制接口 `org.apache.kafka.server.authorizer.Authorizer` 与其默认实现 `kafka.security.authorizer.AclAuthorizer`

通过配置：

``` jproperties
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
```

可以让 kafka 初始化这个类，并在处理请求是进行 acl 判断

> 2.4 之前是 `kafka.security.auth.Authorizer` 接口和 `kafka.security.auth.SimpleAclAuthorizer` 实现类

在 [KafkaServer](/kafka-design/6-src/core/kafka/server/KafkaServer) `startup` 方法中：

``` scala
/* Get the authorizer and initialize it if one is specified.*/
authorizer = config.authorizer
authorizer.foreach(_.configure(config.originals))
val authorizerFutures: Map[Endpoint, CompletableFuture[Void]] = authorizer match {
  case Some(authZ) =>
    authZ.start(brokerInfo.broker.toServerInfo(clusterId, config)).asScala.map { case (ep, cs) =>
      ep -> cs.toCompletableFuture
    }
  case None =>
    brokerInfo.broker.endPoints.map { ep =>
      ep.toJava -> CompletableFuture.completedFuture[Void](null)
    }.toMap
}
```

在 [KafkaApis](/kafka-design/6-src/core/kafka/server/KafkaApis) 中定义：

``` scala
  private[server] def authorize(requestContext: RequestContext,
                                operation: AclOperation,
                                resourceType: ResourceType,
                                resourceName: String,
                                logIfAllowed: Boolean = true,
                                logIfDenied: Boolean = true,
                                refCount: Int = 1): Boolean = {
    authorizer.forall { authZ =>
      val resource = new ResourcePattern(resourceType, resourceName, PatternType.LITERAL)
      val actions = Collections.singletonList(new Action(operation, resource, refCount, logIfAllowed, logIfDenied))
      authZ.authorize(requestContext, actions).get(0) == AuthorizationResult.ALLOWED
    }
  }
```

- `handleProduceRequest`
    - WRITE            TRANSACTIONANL_ID produceRequest.transactionalId
    - IDEMPOTENT_WRITE CLUSTER           CLUSTER_NAME
    - WRITE            TOPIC
- `handleFetchRequest`
    - CLUSTER_ACTION   CLUSTER      CLUSTER_NAME
    - READ TOPIC
- `handleListOffsetRequestV0`
    - DESCRIBE TOPIC
- `handleListOffsetRequestV1AndAbove`
    - DESCRIBE TOPIC

