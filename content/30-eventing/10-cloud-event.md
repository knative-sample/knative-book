---
title: "定义无处不在的事件 - CloudEvent"
date: 2020-5-10T15:26:15Z
draft: false
weight: 10
description: "calling built-in Shortcodes into your content files."
---

## 背景
Event 事件无处不在，然而每个事件提供者产生的事件各部相同。由于缺乏事件的统一描述，对于事件的开发者来说需要不断的重复学习如何消费不同类型的事件。这也限制了类库、工具和基础设施在跨环境（如 SDK、事件路由或跟踪系统）提供事件数据方面的潜力。从事件数据本身实现的可移植性和生产力上受到了阻碍。
## 什么是 CloudEvents
CloudEvents 是一种规范，用于以通用格式描述事件数据，以提供跨服务、平台和系统的交互能力。
事件格式指定了如何使用某些编码格式来序列化 CloudEvent。支持这些编码的兼容 CloudEvents 实现必须遵循在相应的事件格式中指定的编码规则。所有实现都必须支持 JSON 格式。

## 协议规范
### 命名规范
CloudEvents 属性名称必须由 ASCII 字符集的小写字母（“a”到“z”）或数字（“0”到“9”）组成，并且必须以小写字母开头。属性名称应具有描述性和简洁性，长度不应超过20个字符。
### 术语定义
本规范定义如下术语：
- Occurrence： “Occurrence”是指在软件系统运行期间捕获描述信息。
- Event： "Event" 是表示事件及其上下文的数据记录。
- Context:  Context 表示上下文，元数据将封装在 Context 属性中。应用程序代码可以使用这些信息来标识事件与系统或其他事件之间的关系。
- Data： 实际事件中有效信息载荷。
- Message： 事件通过 Message 从数据源传输到目标地址。
- Protocol： 消息可以通过各种行业标准协议（如http、amqp、mqtt、smtp）、开源协议（如kafka、nats）或平台/供应商特定协议（aws-kineis、azure-event-grid）进行传递。

### 上下文属性（Context Attributes）
符合本规范的每个 CloudEvent 必须包括根据需要指定的上下文属性，并且可以包括一个或多个可选的上下文属性。
参考示例：

```yaml
  specversion: 0.2
  type: dev.knative.k8s.event
  source: /apis/serving.knative.dev/v1alpha1/namespaces/default/routes/sls-cloudevent
  id: 269345ff-7d0a-11e9-b1f1-00163f005e02
  time: 2019-05-23T03:23:36Z
  contenttype: application/json
```
- type： 事件类型， 通常此属性用于路由、监控、安全策略等。
- specversion： 表示 CloudEvents 规范的版本。引用 0.2 版本的规范时，事件生产者必须使用 0.2 设置此值。
- source：表示事件的产生者, 也就是事件源。
- id: 事件的 id
- time: 事件的产生时间
- contenttype： 表示`Data` 的数据内容格式
### 扩展属性（Extension Attributes）
CloudEvents 生产者可以在事件中包含其他上下文属性，这些属性可能用于与事件处理相关的辅助操作。
### Data
正如术语`Data`所定义的，CloudEvents 产生具体事件的内容信息封装在数据属性中。例如，KubernetesEventSource所产生的 CloudEvent 的`Data`信息如下：

```yaml
data:
  {
    "metadata": {
      "name": "event-display.15a0a2b54007189b",
      "namespace": "default",
      "selfLink": "/api/v1/namespaces/default/events/event-display.15a0a2b54007189b",
      "uid": "9195ff11-7b9b-11e9-b1f1-00163f005e02",
      "resourceVersion": "18070551",
      "creationTimestamp": "2019-05-21T07:39:30Z"
    },
    "involvedObject": {
      "kind": "Route",
      "namespace": "default",
      "name": "event-display",
      "uid": "31c68419-675b-11e9-a087-00163e08f3bc",
      "apiVersion": "serving.knative.dev/v1alpha1",
      "resourceVersion": "9242540"
    },
    "reason": "InternalError",
    "message": "Operation cannot be fulfilled on clusteringresses.networking.internal.knative.dev \"route-31c68419-675b-11e9-a087-00163e08f3bc\": the object has been modified; please apply your changes to the latest version and try again",
    "source": {
      "component": "route-controller"
    },
    "firstTimestamp": "2019-05-21T07:39:30Z",
    "lastTimestamp": "2019-05-26T07:10:51Z",
    "count": 5636,
    "type": "Warning",
    "eventTime": null,
    "reportingComponent": "",
    "reportingInstance": ""
  }
```
## 实现
以 Go SDK 实现 CloudEvent 0.2 规范为例：
### 事件接收服务
- 导入cloudevents

```go
import "github.com/cloudevents/sdk-go"
```
-通过 HTTP 协议接收 CloudEvent 事件

```go
func Receive(event cloudevents.Event) {
    fmt.Printf("cloudevents.Event\n%s", event.String())
}

func main() {
	c, err := cloudevents.NewDefaultClient()
	if err != nil {
		log.Fatalf("failed to create client, %v", err)
	}
	log.Fatal(c.StartReceiver(context.Background(), Receive));
}
```
### 事件发送服务
- 创建一个基于  0.2 协议的 CloudEvent 事件

```go
event := cloudevents.NewEvent()
event.SetID("ABC-123")
event.SetType("com.cloudevents.readme.sent")
event.SetSource("http://localhost:8080/")
event.SetData(data)
```
- 通过HTTP协议发送这个CloudEvent

```go
t, err := cloudevents.NewHTTPTransport(
	cloudevents.WithTarget("http://localhost:8080/"),
	cloudevents.WithEncoding(cloudevents.HTTPBinaryV02),
)
if err != nil {
	panic("failed to create transport, " + err.Error())
}

c, err := cloudevents.NewClient(t)
if err != nil {
	panic("unable to create cloudevent client: " + err.Error())
}
if err := c.Send(ctx, event); err != nil {
	panic("failed to send cloudevent: " + err.Error())
}
```

这样我们就通过 Go SDK 方式实现了 CloudEvent 事件的发送和接收


