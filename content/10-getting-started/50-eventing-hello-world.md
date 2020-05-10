---
title: "Eventing Hello World"
date: 2020-5-10T15:26:15Z
draft: false
weight: 50
---
基于事件驱动是 Serverless 的核心功能之一，通过事件驱动服务，满足了用户按需付费（Pay-as-you-go）的需求。在之前的文章中我们介绍过 Knative Eventing 由事件源、事件处理模型和事件消费 3 个主要部分构成，那么事件如何通过这 3 个组件产生、处理以及消费呢？ 本文通过 Hello World 示例带你初探 Eventing。

## 前置准备
- Knative 版本 >= 0.5
- 已安装 Serving 组件
- 已安装 Eventing 组件
## 操作步骤
先看一下 Kubernetes Event Source 示例处理流程，如图所示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560157164099-6e1e4416-678a-4e25-b238-05391f60f3d3.png) 
接下来介绍一下各个阶段如何进行操作处理。
### 创建 Service Account
为 `ApiServerSource` 创建 Service Account， 用于授权 ApiServerSource 获取 Kubernetes Events 。
serviceaccount.yaml 如下：
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: events-sa
  namespace: default

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: event-watcher
rules:
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - get
  - list
  - watch

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-ra-event-watcher
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: event-watcher
subjects:
- kind: ServiceAccount
  name: events-sa
  namespace: default
```
执行如下操作：
```
kubectl apply --filename serviceaccount.yaml
```
### 创建 Event Source
Knative Eventing 中 通过 Event Source 对接第三方系统产生统一的事件类型。当前支持 ApiServerSource，GitHub 等多种数据源。这里我们创建一个 ApiServerSource 事件源用于接收 Kubernetes Events 事件并进行转发。k8s-events.yaml 如下：

```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: ApiServerSource
metadata:
  name: testevents
  namespace: default
spec:
  serviceAccountName: events-sa
  mode: Resource
  resources:
  - apiVersion: v1
    kind: Event
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Broker
    name: default
```
这里通过 sink 参数指定事件接收方，支持 Broker 和 k8s service。

执行命令：

```
kubectl apply --filename k8s-events.yaml
```

### 创建 Knative Service
首先构建你的事件处理服务，可以参考 [knative-sample/event-display](https://github.com/knative-sample/event-display)开源项目。
这里的 Service 服务仅把接收到的事件打印出来，处理逻辑如下：

```go
package main
import (
	"context"
	"fmt"
	"log"
	cloudevents "github.com/cloudevents/sdk-go"
	"github.com/knative-sample/event-display/pkg/kncloudevents"
)
/*
Example Output:

☁  cloudevents.Event:
Validation: valid
Context Attributes,
  SpecVersion: 0.2
  Type: dev.knative.eventing.samples.heartbeat
  Source: https://github.com/knative/eventing-sources/cmd/heartbeats/#local/demo
  ID: 3d2b5a1f-10ca-437b-a374-9c49e43c02fb
  Time: 2019-03-14T21:21:29.366002Z
  ContentType: application/json
  Extensions:
    the: 42
    beats: true
    heart: yes
Transport Context,
  URI: /
  Host: localhost:8080
  Method: POST
Data
  {
    "id":162,
    "label":""
  }
*/

func display(event cloudevents.Event) {
	fmt.Printf("Hello World: \n")
	fmt.Printf("cloudevents.Event\n%s", event.String())
}

func main() {
	c, err := kncloudevents.NewDefaultClient()
	if err != nil {
		log.Fatal("Failed to create client, ", err)
	}
	log.Fatal(c.StartReceiver(context.Background(), display))
}

```
通过上面的代码，可以轻松构建你自己的镜像。镜像构建完成之后，接下来可以创建一个简单的 Knative Service， 用于消费 `ApiServerSource` 产生的事件。
service.yaml 示例如下：

```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: event-display
  namespace: default
spec:
  template:
    spec:
      containers:
      -  image: {yourrepo}/{yournamespace}/event-display:latest
```
执行命令：

```
kubectl apply --filename service.yaml
```

### 创建 Broker
在所选命名空间下，创建 `default` Broker。假如选择 `default` 命名空间， 执行操作如下。

```
kubectl label namespace default knative-eventing-injection=enabled
```
这里 Eventing Controller 会根据设置`knative-eventing-injection=enabled` 标签的 namepace， 自动创建 Broker。并且使用在webhook中默认配置的 ClusterChannelProvisioner（in-memory）。


### 创建 Trigger
Trigger 可以理解为 Broker 和Service 之间的过滤器，可以设置一些事件的过滤规则。这里为默认的 Broker 创建一个最简单的 Trigger，并且使用 Service 进行订阅。trigger.yaml 示例如下：
```
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: testevents-trigger
  namespace: default
spec:
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: event-display
```
执行命令：

```
kubectl apply --filename trigger.yaml
```

注意：如果没有使用默认的 Broker, 在 Trigger 中可以通过 spec.broker 指定 Broker 名称。

## 验证
执行如下命令，生成 k8s events。
```
kubectl run busybox --image=busybox --restart=Never -- ls
kubectl delete pod busybox
```

可以通过下述方式查看 Knative Service 是否接收到事件。
```
kubectl get pods
kubectl logs -l serving.knative.dev/service=event-display -c user-container
```
日志输出类似下面，说明已经成功接收事件。

```
Hello World:  
☁️  CloudEvent: valid ✅
Context Attributes,
  SpecVersion: 0.2
  Type: dev.knative.apiserver.resource.add
  Source: https://10.39.240.1:443
  ID: 716d4536-3b92-4fbb-98d9-14bfcf94683f
  Time: 2019-05-10T23:27:06.695575294Z
  ContentType: application/json
  Extensions:
    knativehistory: default-broker-b7k2p-channel-z7mqq.default.svc.cluster.local
    subject: /apis/v1/namespaces/default/events/busybox.159d7608e3a3572c
Transport Context,
  URI: /
  Host: auto-event-display.default.svc.cluster.local
  Method: POST
Data,
  {
    "apiVersion": "v1",
    "count": 1,
    "eventTime": null,
    "firstTimestamp": "2019-05-10T23:27:06Z",
    "involvedObject": {
      "apiVersion": "v1",
      "fieldPath": "spec.containers{busybox}",
      "kind": "Pod",
      "name": "busybox",
      "namespace": "default",
      "resourceVersion": "28987493",
      "uid": "1efb342a-737b-11e9-a6c5-42010a8a00ed"
    },
    "kind": "Event",
    "lastTimestamp": "2019-05-10T23:27:06Z",
    "message": "Started container",
    "metadata": {
      "creationTimestamp": "2019-05-10T23:27:06Z",
      "name": "busybox.159d7608e3a3572c",
      "namespace": "default",
      "resourceVersion": "506088",
      "selfLink": "/api/v1/namespaces/default/events/busybox.159d7608e3a3572c",
      "uid": "2005af47-737b-11e9-a6c5-42010a8a00ed"
    },
    "reason": "Started",
    "reportingComponent": "",
    "reportingInstance": "",
    "source": {
      "component": "kubelet",
      "host": "gke-knative-auto-cluster-default-pool-23c23c4f-xdj0"
    },
    "type": "Normal"
  }
```
## 总结
相信通过上面的例子你已经了解了 Knative Eventing 如何产生事件、处理事件以及消费事件。当然你可以自己定义一个事件消费服务，来处理事件。


