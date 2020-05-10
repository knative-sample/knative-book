---
title: "Sequeue 解析"
date: 2020-5-10T15:26:15Z
draft: false
weight: 1000
description: "calling built-in Shortcodes into your content files."
---


在实际的开发中我们会经常遇到将一条数据需要经过多次处理的场景，称为 Pipeline。那么在 Knative 中是否也提供这样的能力呢？其实从 Knative Eventing 0.7 版本开始，提供了 Sequence CRD 资源，用于事件处理 Pipeline。下面我们介绍一下 Sequence。
## Sequence 定义
首先我们看一下 Sequence Spec 定义：

```
apiVersion: messaging.knative.dev/v1alpha1
kind: Sequence
metadata:
  name: test
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
  steps:
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: test
  reply:
    kind: Broker
    apiVersion: eventing.knative.dev/v1alpha1
    name: test
```
Sequence Spec包括3个部分：
1. steps： 在 step 中定义了按照顺序执行的服务，每个服务会对应创建 Subscription
2. channelTemplate：指定了使用具体的那个 Channel
3. reply：（可选）定义了将最后一个 step 服务结果转发到的目标服务

Sequence 都是适合哪些具体应用场景呢？我们上面也提到了事件处理的 Pipeline。那么在实际场景应用中究竟以什么样的形式体现呢？ 现在我们揭晓一下 Sequence 在 Knative Eventing 中提供的如下 4 种使用场景：
- 直接访问 Service
- 面向事件处理
- 级联 Sequence
- 面向 Broker/Trigger

## 直接访问 Service 场景
事件源产生的事件直接发送给 Sequence 服务， Sequence 接收到事件之后顺序调用 Service 服务对事件进行处理：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566301044661-b5dd39b1-b1a1-40b6-98f3-8b2fefe3a85e.png) 
### 创建 Knative Service
这里我们创建 3 个 Knative Service 用于事件处理。每个 Service 接收到事件之后会打印当前的事件处理信息。
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: first
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "0"

---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: second
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "1"
---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: third
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "2"
---
```
### 创建 Sequence
创建顺序调用 `first->second->third` Service 的 Sequence。
```
apiVersion: messaging.knative.dev/v1alpha1
kind: Sequence
metadata:
  name: sequence
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
  steps:
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: first
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: second
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: third
```
### 创建数据源
创建 CronJobSource 数据源，每隔 1 分钟发送一条事件消息`{"message": "Hello world!"}`到 Sequence 服务。
```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CronJobSource
metadata:
  name: cronjob-source
spec:
  schedule: "*/1 * * * *"
  data: '{"message": "Hello world!"}'
  sink:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: Sequence
    name: sequence
```
### 示例结果
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566438724005-97250a1c-4ad0-4cd4-bec7-d8b08321767c.png) 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566438579343-e2b01c05-3599-4965-a37c-5e9b5772f5f1.png) 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566438608300-c958e006-83e9-4292-ab35-5fc5fc9a7eca.png) 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566438642722-852bf289-1f07-46bb-9bca-eb32f1426aee.png) 
## 面向事件处理场景
事件源产生的事件直接发送给 Sequence 服务， Sequence 接收到事件之后顺序调用 Service 服务对事件进行处理，处理之后的最终结果会调用`event-display` Service 显示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566301086528-1e8c2ed2-9953-401d-86f2-ac2ffd1f2722.png) 
### 创建 Knative Service
同上创建 3 个 Knative Service 用于事件处理：
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: first
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "0"

---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: second
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "1"
---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: third
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "2"
---
```
### 创建 Sequence
创建顺序调用 `first->second->third` Service 的 Sequence，将处理结果通过`reply`发送给`event-display`
```
apiVersion: messaging.knative.dev/v1alpha1
kind: Sequence
metadata:
  name: sequence
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
  steps:
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: first
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: second
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: third
  reply:
    kind: Service
    apiVersion: serving.knative.dev/v1alpha1
    name: event-display
```
### 创建结果显示 Service
创建`event-display` Service, 用于接收最终的结果信息。
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: event-display
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-release/event_display:bf45b3eb1e7fc4cb63d6a5a6416cf696295484a7662e0cf9ccdf5c080542c21d
```
### 创建数据源
创建 CronJobSource 数据源，每隔 1 分钟发送一条事件消息`{"message": "Hello world!"}`到 Sequence 服务。
```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CronJobSource
metadata:
  name: cronjob-source
spec:
  schedule: "*/1 * * * *"
  data: '{"message": "Hello world!"}'
  sink:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: Sequence
    name: sequence
```
### 示例结果
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566440146782-069c1846-eb9b-480b-96df-fb3a114e8259.png) 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566440197847-82c36ac6-c4fd-424b-8308-367d47d32a8c.png) 
## 级联 Sequence 场景
Sequence 更高级的地方还在于支持级联处理: Sequence By Sequence，这样可以进行多次 Sequence 处理，满足复杂事件处理场景需求：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566301158416-740d7807-c13f-42c6-80d4-907f5d0aaf1d.png) 
### 创建 Knative Service
创建 6 个 Knative Service 用于事件处理， 前 3 个用于第 1 个 Sequence，后 3 个用于第 2 个 Sequence。
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: first
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "0"

---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: second
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "1"
---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: third
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "2"
---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: fourth
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "3"

---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: fifth
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "4"
---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: sixth
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
          env:
            - name: STEP
              value: "5"
---
```
### 创建第 1 个 Sequence
使用 `first->second->third` Service 用于第 1 个 Sequence 调用处理，将执行结果发送给第 2 个 Sequence。
```
apiVersion: messaging.knative.dev/v1alpha1
kind: Sequence
metadata:
  name: first-sequence
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
  steps:
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: first
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: second
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: third
  reply:
    kind: Sequence
    apiVersion: messaging.knative.dev/v1alpha1
    name: second-sequence
```
### 创建第 2 个 Sequence
使用 `fourth->fifth->sixth` Service 用于第 2 个 Sequence 调用处理，将执行结果发送给 `event-display`。
```
apiVersion: messaging.knative.dev/v1alpha1
kind: Sequence
metadata:
  name: second-sequence
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
  steps:
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: fourth
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: fifth
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: sixth
  reply:
    kind: Service
    apiVersion: serving.knative.dev/v1alpha1
    name: event-display
```
### 创建结果显示 Service
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: event-display
spec:
  template:
    spec:
      containerers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-release/event_display:bf45b3eb1e7fc4cb63d6a5a6416cf696295484a7662e0cf9ccdf5c080542c21d
```
### 创建数据源指向第 1 个 Sequence
```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CronJobSource
metadata:
  name: cronjob-source
spec:
  schedule: "*/1 * * * *"
  data: '{"message": "Hello world!"}'
  sink:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: Sequence
    name: first-sequence
```
### 示例结果
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566441093339-6b8ada07-8597-409b-a21c-e864a831c2bd.png) 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566441131307-1918c419-3579-4d82-b583-ad69efe61e1f.png) 
## Broker/Trigger 场景
事件源 cronjobsource 向 Broker 发送事件，通过Trigger 将这些事件发送到由 3 个 Service 调用的 Sequence 中。Sequence 处理完之后将结果事件发送给 Broker，并最终由另一个Trigger发送给 `event-display` Service 显示事件结果:
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1564667214555-480f8210-0623-4b94-83b6-a393b6753d1e.png) 

### 创建 Knative Service
同上创建 3 个 Knative Service，用于 Sequence 中服务处理。

```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: first
spec:
  template:
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
        env:
        - name: STEP
          value: "0"

---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: second
spec:
  template:
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
        env:
        - name: STEP
          value: "1"
---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: third
spec:
  template:
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/probable-summer:2656f39a7fcb6afd9fc79e7a4e215d14d651dc674f38020d1d18c6f04b220700
        env:
        - name: STEP
          value: "2"

---
```

### 创建 Sequence
创建 Sequence，这里依次顺序执行`first->second->third`这3个服务。将最终处理的结果发送到`broker-test`中。
```
apiVersion: messaging.knative.dev/v1alpha1
kind: Sequence
metadata:
  name: sequence
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
  steps:
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: first
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: second
    - ref:
        apiVersion: serving.knative.dev/v1alpha1
        kind: Service
        name: third
  reply:
    kind: Broker
    apiVersion: eventing.knative.dev/v1alpha1
    name: default
```

### 创建事件源指向 Broker
创建 CronjobSource，它将每隔 1 分钟发送一条`{"message": "Hello world!"} `消息到 broker-test 中。

```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CronJobSource
metadata:
  name: cronjob-source
spec:
  schedule: "*/1 * * * *"
  data: '{"message": "Hello world!"}'
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Broker
    name: default
```
### 创建 Broker
创建默认 Broker
```
kubectl label namespace default knative-eventing-injection=enabled
```
### 创建 Trigger  指向 Sequence
创建订阅事件类型为`dev.knative.cronjob.event`的 Trigger, 用于 Sequence 进行消费处理。
```
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: sequence-trigger
spec:
  filter:
    sourceAndType:
      type: dev.knative.cronjob.event
  subscriber:
    ref:
      apiVersion: messaging.knative.dev/v1alpha1
      kind: Sequence
      name: sequence
```
### 创建结果订阅 Trigger
创建订阅 `samples.http.mod3` 的事件类型 Trigger，将 Sequence 执行的结果发送给`event-display` Service 进行显示。
```
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: display-trigger
spec:
  filter:
    sourceAndType:
      type: samples.http.mod3
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: event-display
---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: event-display
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-release/event_display:bf45b3eb1e7fc4cb63d6a5a6416cf696295484a7662e0cf9ccdf5c080542c21d
---
```
### 示例结果
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566443862869-586cf09f-5e92-4a47-86c3-bdc5b4151414.png) 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566443884993-2f24395b-b2ac-43d6-b385-f80ab0d30f7f.png) 
## 总结
以上介绍了什么是 Sequence， 以及基于 Sequence 的 4 种使用场景，我们可以根据实际需求选择不同的使用场景，从而实现事件处理 Pipeline。这对于需要多步骤处理事件的场景尤为适合。


