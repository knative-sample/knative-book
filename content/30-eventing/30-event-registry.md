---
title: "事件注册机制 - Registry"
date: 2020-5-10T15:26:15Z
draft: false
weight: 30
description: "calling built-in Shortcodes into your content files."
---

## 背景
作为事件消费者，之前是无法事先知道哪些事件可以被消费，如果能通过某种方式获得哪些 Broker 提供哪些事件,那么事件消费者就能很方便通过这些 Broker 消费事件。Registry 就是在这样的背景下被提出的，通过 Registry 机制，消费者能针对特定的 Broker 的事件通过 Trigger 进行事件订阅消费。这里需要说明一下，Registry 设计与实现目前是针对 Broker/Trigger 事件处理模型。 
## 诉求
- 每个Registry 对应一个 Namespace 作为资源隔离的边界
- Registry 中包括事件类型列表，以提供事件发现机制，通过事件列表，我们可以决定对哪些 Ready 的事件进行消费
- 由于事件最终需要通过 Trigger 进行订阅，因此事件类型信息中需要包括创建 Trigger 所需要的信息。

## 实现
### 定义 EventType CRD 资源
定义 EventType 类型的CRD资源，示例yaml：
```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: EventType
metadata:
  name: com.github.pullrequest
  namespace: default
spec:
  type: com.github.pull_request
  source: github.com
  schema: //github.com/schemas/pull_request
  description: "GitHub pull request"
  broker: default
```
- name: 设置时需要符合k8s命名规范
- type：遵循CloudEvent 类型设置。
- source: 遵循 CloudEvent source设置。
- schema: 可以是JSON schema, protobuf schema等。可选项。
- description: 事件描述，可选项。
- broker: 所提供 EventType 的 Broker。

### 事件注册与发现
1.创建 EventType
一般我们通过以下两种方式创建 EventType 实现事件的注册。
1.1通过Event 事件源实例自动注册
基于事件源实例，通过事件源控制器创建 EventType 进行注册：

```yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: GitHubSource
metadata:
  name: github-source-sample
  namespace: default
spec:
  eventTypes:
    - push
    - pull_request
  ownerAndRepository: my-other-user/my-other-repo
  accessToken:
    secretKeyRef:
      name: github-secret
      key: accessToken
  secretToken:
    secretKeyRef:
      name: github-secret
      key: secretToken
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Broker
    name: default
```
通过运行上面的yaml信息，通过GitHubSource 事件源控制器可以创建事件类型dev.knative.source.github.push以及事件类型dev.knative.source.github.pull_request这两个EventType进行注册，其中source为github.com以及名称为default的Broker。具体如下：

```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: EventType
metadata:
  generateName: dev.knative.source.github.push-
  namespace: default
  owner: # Owned by github-source-sample
spec:
  type: dev.knative.source.github.push
  source: github.com
  broker: default
---
apiVersion: eventing.knative.dev/v1alpha1
kind: EventType
metadata:
  generateName: dev.knative.source.github.pullrequest-
  namespace: default
  owner: # Owned by github-source-sample
spec:
  type: dev.knative.source.github.pull_request
  source: github.com
  broker: default
```
这里有两点需要注意：
- 通过自动生成默认名称，避免名称冲突。对于是否应该在代码（在本例中是GithubSource控制器）创建事件类型时生成，需要进一步讨论。
- 我们给`spec.type`加上了`dev.knative.source.github.`前缀，这个也需要进一步讨论确定是否合理。

1.2手动注册
直接通过创建EventType CR资源实现注册，如：

```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: EventType
metadata:
  name: org.bitbucket.repofork
  namespace: default
spec:
  type: org.bitbucket.repo:fork
  source: bitbucket.org
  broker: dev
  description: "BitBucket fork"
```
通过这种方式可以注册名称为`org.bitbucket.repofork`, type 为 `org.bitbucket.repo:fork`, source 为 `bitbucket.org` 以及属于`dev` Broker 的 EventType。

2.获取 Registry 的事件注册
事件消费者可以通过如下方式获取 Registry 的事件注册列表：
$ kubectl get eventtypes -n default

```
NAME                                         TYPE                                    SOURCE          SCHEMA                              BROKER     DESCRIPTION           READY   REASON
org.bitbucket.repofork                       org.bitbucket.repo:fork                 bitbucket.org                                       dev        BitBucket fork        False   BrokerIsNotReady
com.github.pullrequest                       com.github.pull_request                 github.com      //github.com/schemas/pull_request   default    GitHub pull request   True 
dev.knative.source.github.push-34cnb         dev.knative.source.github.push          github.com                                          default                          True 
dev.knative.source.github.pullrequest-86jhv  dev.knative.source.github.pull_request  github.com                                          default                          True  
```

3.Trigger 订阅事件
最后基于事件注册列表信息，事件消费者创建对应的Trigger对Registry中的EventType进行监听消费
Trigger示例：

```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: my-service-trigger
  namespace: default
spec:
  filter:
    sourceAndType:
      type: dev.knative.source.github.push
      source: github.com
  subscriber:
    ref:
     apiVersion: serving.knative.dev/v1alpha1
     kind: Service
     name: my-service
```
## 总结
Registry 的设计主要是针对 Broker/Trigger 事件处理模型。创建事件源资源时，会创建EventType注册到Registry。在实现方面，我们可以检查Event Source的Sink类型是否是Broker，如果是，则对其注册EventType。

