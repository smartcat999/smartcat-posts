---
title: "KafkaOperator配置"
draft: false
toc: true
description: "KafkaOperator基于connector/user/topic的管理配置"
featured_image: /images/bg02.jpg
categories: []
tags: [kafka, operator]
---
Kafka Operator相关配置介绍<!--more-->
#### 1 kafka connector管理
##### 1.1 创建kafka connector
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect-cluster  # connector名字
  annotations:
    strimzi.io/use-connector-resources: "true" # 是否使用connector
spec:
  replicas: 3 # 副本数
  authentication: # connect使用的认证方式
    type: tls
    certificateAndKey:
      certificate: source.crt
      key: source.key
      secretName: my-user-source
  bootstrapServers: my-cluster-kafka-bootstrap:9092 # kafka的server地址
  tls: 
    trustedCertificates:
      - secretName: my-cluster-cluster-cert  # 连接集群所用secret证书的名字
        certificate: ca.crt
  config: # connect配置
    group.id: my-connect-cluster
    offset.storage.topic: my-connect-cluster-offsets
    config.storage.topic: my-connect-cluster-configs
    status.storage.topic: my-connect-cluster-status
    key.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: true
    value.converter.schemas.enable: true
    config.storage.replication.factor: 3
    offset.storage.replication.factor: 3
    status.storage.replication.factor: 3
  resources: # pod的资源配置
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      cpu: "2"
      memory: 2Gi
  logging: # 日志配置
    type: inline
    loggers:
      log4j.rootLogger: "INFO"
  readinessProbe: # pod是否read检测
    initialDelaySeconds: 15
    timeoutSeconds: 5
  livenessProbe: # pod是否存活检测
    initialDelaySeconds: 15
    timeoutSeconds: 5
  jvmOptions: # 运行jvm配置
    "-Xmx": "1g"
    "-Xms": "1g"
  template: # pod的template配置
    pod:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: application
                    operator: In
                    values:
                      - postgresql
                      - mongodb
              topologyKey: "kubernetes.io/hostname"
    connectContainer: # Jaeger tracer配置
      env:
        - name: JAEGER_SERVICE_NAME
          value: my-jaeger-service
        - name: JAEGER_AGENT_HOST
          value: jaeger-agent-name
        - name: JAEGER_AGENT_PORT
          value: "6831"
```

说明：

* tls字段表示连接broker集群所有的证书的secret，secret的默认名字是{CLUSTER-NAME}-cluster-cert，authentication表示connect使用的认证方式，用来连接connect的。

##### 1.2 修改connect配置

只需要修改spec.config字段，既可以修改connect的配置

```yaml
spec:
  config: 
    group.id: my-connect-cluster
    offset.storage.topic: my-connect-cluster-offsets
    config.storage.topic: my-connect-cluster-configs
    status.storage.topic: my-connect-cluster-status
    key.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: true
    value.converter.schemas.enable: true
    config.storage.replication.factor: 3
    offset.storage.replication.factor: 3
    status.storage.replication.factor: 3
```

config的具体配置参见https://kafka.apache.org/documentation/#connectconfigs

##### 1.3 删除connect

删除connect对应的cr资源即可

```shell
kubectl delete kafkaconnect CONNECT-NAME -n NAMESPACE
```

#### 2 user管理

##### 2.1 创建user资源

kafkauser资源只有在认证模式为scram-sha-512或者tls时才会生效

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser 
metadata:
  name: saas  # 用户名称
  labels:
    strimzi.io/cluster: my-cluster  # kafka集群名字
spec:
  authentication:
    type: scram-sha-512  # 认证方式，和集群中端口的认证方式对应
  authorization:
    type: simple  # 授权方式，采用默认值即可
    acls:
      # Example consumer Acls for topic my-topic using consumer group my-group
      - resource:  # 资源做权限控制
          type: topic  #  资源类型，存在多种资源类型，参看https://wiki.yunify.com/pages/viewpage.action?pageId=128423310
          name: my-topic # 资源名称
          patternType: literal # 匹配模式
        operation: Read # 权限
        host: "*" # 作用的ip
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operation: Describe
        host: "*"
      - resource:
          type: group
          name: my-group
          patternType: literal
        operation: Read
        host: "*"
      # Example Producer Acls for topic my-topic
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operation: Write
        host: "*"
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operation: Create
        host: "*"
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operation: Describe
        host: "*"
```

##### 2.2 修改user资源中acl

修改spec.authorization.acls字段即可

```yaml
spec:
  authorization:
    acls:
      # Example consumer Acls for topic my-topic using consumer group my-group
      - resource:  # 资源做权限控制
          type: topic  #  资源类型，存在多种资源类型，参看https://kafka.apache.org/documentation/#security_authz_cli
          name: my-topic # 资源名称
          patternType: literal # 匹配模式
        operation: Read # 权限
        host: "*" # 作用的ip
```

##### 2.3 修改user资源中的密码

修改密码的操作，只能在当认证方式为scram-sha-512时，通过创建自定义的secret，然后按照下面方式将值传递给password。注意secret中关于的密码的值需要经过base64编码。

```yaml
spec:
  authentication:
    type: scram-sha-512
    password:
      valueFrom:
        secretKeyRef:
          name: my-secret 
          key: my-password 
```

##### 2.4 查看对应用户的信息

创建用户之后，在当前namespace下面会出现对应用户的secret，查看对应的secret即可。其中password这一行的值是经过base64编码之后的结果。

##删除用户资源

直接删除对应的cr资源即可

```shell
kubectl delete kafkauser USER_NAME -n NAMESPACE
```

#### 3 kafka topic管理

##### 3.1 创建topic资源

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic2  # topic名字
  labels:
    strimzi.io/cluster: "my-cluster"  # 对应集群名字
spec:
  partitions: 3  # partition的数目
  replicas: 1 # 副本数目
  config: # topic的其它配置
    retention.ms: 7200000
    segment.bytes: 1073741824
```

##### 3.2 修改topic资源

修改 spec →config、spec →partitions和spec→replicas

```yaml
spec:
 partitions: 3  # 分区数
 replicas: 1  # 副本数
 config:  # 其它参数配置
   retention.ms: 7200000
   segment.bytes: 1073741824
```

config中的具体参数值可以参见https://kafka.apache.org/documentation/#topicconfigs

##### 3.3 删除topic资源

直接删除对应cr资源即可，如下命令

```shell
kebectl delete kafkatopic TOPIC_NAME -n NAMESPACE
```


