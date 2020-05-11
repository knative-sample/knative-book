---
title: "基于 Kafka 实现消息推送"
date: 2020-5-10T15:26:15Z
draft: false
weight: 40
description: ""
---

当前在 Knative 中已经提供了对 Kafka 事件源的支持，那么如何基于 Kafka 实现消息推送，本文以阿里云 Kafka 产品为例，给大家解锁这一新的姿势。
## 背景
消息队列 for Apache Kafka 是阿里云提供的分布式、高吞吐、可扩展的消息队列服务。消息队列 for Apache Kafka 广泛用于日志收集、监控数据聚合、流式数据处理、在线和离线分析等大数据领域，已成为大数据生态中不可或缺的部分。
结合 Knative  中提供了 KafkaSource 事件源的支持， 可以方便的对接 Kafka 消息服务。
另外也可以安装社区 Kafka 2.0.0 及以上版本使用。
## 在阿里云上创建 Kafka 实例
### 创建 Kafka 实例
登录[消息队列Kafka控制台](https://kafka.console.aliyun.com), 选择【购买实例】。由于当前 Knative 中 Kafka 事件源支持 2.0.0 及以上版本，在阿里云上创建 Kafka 实例需要选择包年包月、专业版本进行购买，购买之后升级到 2.0.0 即可。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1571112073879-30b47c28-adb7-4cf2-b92d-cd933a5f60c0.png) 
### 部署实例并绑定VPC
购买完成之后，进行部署，部署时设置 Knative 集群所在的 VPC 即可：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1571119132823-16cfe979-aae7-4175-9225-40a53ad552a1.png) 

### 创建 Topic 和 Consumer Group
接下来我们创建 Topic 和消费组。
进入【Topic 管理】，点击`创建Topic`, 这里我们创建名称为 demo 的 topic:
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1571142754737-404b00dc-af46-47c1-91c1-505f1870f9a1.png) 
进入【Consumer Group 管理】,点击`创建 Consumer Group`, 这里我们创建名称为 demo-consumer 的消费组:
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1571142852151-184151d7-bd32-4d7d-87fa-2b437d06bedd.png) 

## 部署 Kafka 数据源
### 部署 Kafka addon 组件
登录容器服务控制台，进入【Knative 组件管理】，部署 Kafka addon 组件。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572871446071-ac71fdaf-8ae1-4d71-b86d-403e02aeb3e6.png) 

### 创建 KafkaSource 实例
首先创建用于接收事件的服务 event-display:
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: event-display
spec:
  template:
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/eventing-sources-cmd-event_display:bf45b3eb1e7fc4cb63d6a5a6416cf696295484a7662e0cf9ccdf5c080542c21d
```
接下来创建KafkaSource
```yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: KafkaSource
metadata:
  name: alikafka-source
spec:
  consumerGroup: demo-consumer
  # Broker URL. Replace this with the URLs for your kafka cluster,
  # which is in the format of my-cluster-kafka-bootstrap.my-kafka-namespace:9092.
  bootstrapServers: 192.168.0.6x:9092,192.168.0.7x:9092,192.168.0.8x:9092
  topics: demo
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: event-display
```
说明：
- bootstrapServers: Kafka VPC 访问地址
- consumerGroup： 设置消费组
- topics：设置 Topic


创建完成之后，我们可以查看对应的实例已经运行：

```bash
[root@iZ2zeae8wzyq0ypgjowzq2Z ~]# kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
alikafka-source-k22vz-db44cc7f8-879pj   1/1     Running   0          8h
```

## 验证
在 Kafka 控制台，选择 topic 发送消息，注意这里的消息格式必须是 json 格式：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1571143441456-2a028f6a-7ec9-4b85-9e87-26b6289c5279.png) 
我们可以看到已经接收到了发送过来的 Kafka 消息：

```bash
[root@iZ2zeae8wzyq0ypgjowzq2Z ~]# kubectl logs event-display-zl6m5-deployment-6bf9596b4f-8psx4 user-container

☁️  CloudEvent: valid ✅
Context Attributes,
  SpecVersion: 0.2
  Type: dev.knative.kafka.event
  Source: /apis/v1/namespaces/default/kafkasources/alikafka-source#demo
  ID: partition:7/offset:1
  Time: 2019-10-18T08:50:32.492Z
  ContentType: application/json
  Extensions: 
    key: demo
Transport Context,
  URI: /
  Host: event-display.default.svc.cluster.local
  Method: POST
Data,
  {
    "key": "test"
  }
```

## 总结
结合阿里云 Kafka 产品，通过事件驱动触发服务（函数）执行，是不是简单又高效。这样我们利用 Knative 得以把云原生的能力充分释放出来，带给我们更多的想象空间。欢迎对 Knative 感兴趣的一起交流。
