---
title: "初识 Knative"
date: 2020-5-10T15:26:15Z
draft: false
weight: 10
---

## Knative 是什么
Knative 是 Google 在 2018 的 Google Cloud Next 大会上发布的一款基于 Kubernetes 的 Serverless 框架。Knative 一个很重要的目标就是制定云原生、跨平台的 Serverless 编排标准。Knative 是通过整合容器构建(或者函数)、工作负载管理(和动态扩缩)以及事件模型这三者来实现的这一 Serverless 标准。Knative 社区的主要贡献者有 Google、Pivotal、IBM、Red Hat。可见其阵容强大， CloudFoundry、OpenShift 这些 PAAS 提供商都在积极的参与 Knative 的建设。

## Knative 出现的背景
在 Knative 之前社区已经有很多 Serverless 解决方案，如下所示这些：
- kubeless
- Fission 
- OpenFaaS 
- Apache OpenWhisk
- ...

除了上面这些社区的开源解决方案以外各大云厂商也都有各自的 FAAS 产品的实现比如：
- AWS Lambda
- Google Cloud Functions
- Microsoft Azure Functions
- 阿里云的函数计算

业务代码部署到 Serverless 平台上就离不开源码的编译、部署和事件的管理。然而无论是开源的解决方案还是各公有云的 FAAS 产品大家的实现方式大家都各不相同，缺乏统一的标准导致市场呈现碎片化。因此无论选择哪一个方案都面临供应商绑定的风险。

没有统一的标准、市场的碎片化这对云厂商来说用户 Serverless 上云就比较困难，对于 PAAS 提供商来说很难做一个通用的 PAAS 平台给用户使用。基于这样的背景 Google 牵头联合 Pivotal、IBM、Red Hat 等发起了 Knative 项目。

我们看一下在 Knative 体系下各个角色的协作关系：

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1558497066382-e59c65f6-65da-4627-aa52-d6ff17e67551.png) 

- Developers
   Serverless 服务的开发人员可以直接使用原生的 Kubernetes API 基于 Knative 部署 Serverless 服务
- Contributors
   主要是指社区的贡献者
- Operators
  Knative 可以被集成到任何支持的环境中，比如：云厂商、或者企业内部。目前 Knative 是基于 Kubernetes 来实现的，有 Kubernetes 的地方就可以部署 Knative
- Users
  终端用户通过 Istio 网关访问服务，或者通过事件系统触发 Knative 中的 Serverless 服务
  
## 核心组件 
Knative 主要由 Serving 和 Eventing 核心组件构成。除此之外使用 Tekton 作为CI/CD构建工具。下面让我们来分别介绍一下这三个核心组件。
### Serving
Knative 作为 Severless 框架最终是用来提供服务的， 那么 Knative Serving 应运而生。
Knative Serving 构建于 Kubernetes 和 Istio 之上，为  Serverless 应用提供部署和服务支持。其特性如下：
- 快速部署 Serverless 容器
- 支持自动扩缩容和缩到 0 实例
- 基于 Istio 组件，提供路由和网络编程
- 支持部署快照

Knative Serving 中定义了以下 CRD 资源：
- Service: 自动管理工作负载整个生命周期。负责创建 Route、Configuration 以及 Revision 资源。通过 Service 可以指定路由流量使用最新的 Revision 还是固定的 Revision
- Route：负责映射网络端点到一个或多个 Revision。可以通过多种方式管理流量。包括灰度流量和重命名路由
- Configuration: 负责保持 Deployment 的期望状态，提供了代码和配置之间清晰的分离，并遵循应用开发的 12 要素。修改一次 Configuration 产生一个 Revision
- Revision：Revision 资源是对工作负载进行的每个修改的代码和配置的时间点快照。Revision 是不可变对象，可以长期保留

资源关系图：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1551788277363-60c608df-37e4-4fa3-9b28-f048ff6f3c64.png) 
## Eventing
Knative Eventing 旨在满足云原生开发中通用需求, 以提供可组合的方式绑定事件源和事件消费者。其设计目标：
-  Knative Eventing 提供的服务是松散耦合，可独立开发和部署。服务可跨平台使用（如 Kubernetes, VMs, SaaS 或者 FaaS）
- 事件的生产者和事件的消费者是相互独立的。任何事件的生产者（事件源）可以先于事件的消费者监听之前产生事件，同样事件的消费者可以先于事件产生之前监听事件
- 支持第三方的服务对接该 Eventing 系统
- 确保跨服务的互操作性

事件处理示意图：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1558667976689-1b1291f1-f82d-4345-80f1-0429ef5994ed.png) 

如上图所示，Eventing 主要由事件源（Event Source）、事件处理（Flow）以及事件消费者（Event Consumer）三部分构成。
### 事件源（Event Source）
当前支持以下几种类型的事件源：
- ApiserverSource：每次创建或更新 Kubernetes 资源时，ApiserverSource 都会触发一个新事件
- GitHubSource：GitHub 操作时，GitHubSource 会触发一个新事件
- GcpPubSubSource： GCP 云平台 Pub/Sub 服务会触发一个新事件
- AwsSqsSource：Aws 云平台 SQS 服务会触发一个新事件
- ContainerSource: ContainerSource 将实例化一个容器，通过该容器产生事件
- CronJobSource: 通过 CronJob 产生事件
- KafkaSource: 接收 Kafka 事件并触发一个新事件
- CamelSource: 接收 Camel 相关组件事件并触发一个新事件

### 事件接收/转发（Flow）
当前 Knative 支持如下事件接收处理：
- 直接事件接收
通过事件源直接转发到单一事件消费者。支持直接调用 Knative Service 或者 Kubernetes Service 进行消费处理。这样的场景下，如果调用的服务不可用，事件源负责重试机制处理。
- 通过事件通道(Channel)以及事件订阅(Subscriptions)转发事件处理
这样的情况下，可以通过 Channel 保证事件不丢失并进行缓冲处理，通过 Subscriptions 订阅事件以满足多个消费端处理。
- 通过 brokers 和 triggers 支持事件消费及过滤机制

从 v0.5 开始，Knative Eventing 定义 Broker 和 Trigger 对象，实现了对事件进行过滤（亦如通过 ingress 和 ingress controller 对网络流量的过滤一样）。
通过定义 Broker 创建 Channel，通过 Trigger 创建 Channel 的订阅（subscription），并产生事件过滤规则。
### 事件消费者（Event Consumer）
为了满足将事件发送到不同类型的服务进行消费， Knative Eventing 通过多个 k8s 资源定义了两个通用的接口：
- Addressable 接口提供可用于事件接收和发送的 HTTP 请求地址，并通过`status.address.hostname`字段定义。作为一种特殊情况，Kubernetes Service 对象也可以实现 Addressable 接口
- Callable 接口接收通过 HTTP 传递的事件并转换事件。可以按照处理来自外部事件源事件的相同方式，对这些返回的事件做进一步处理

当前 Knative 支持通过 Knative Service 或者 Kubernetes Service 进行消费事件。
另外针对事件消费者，如何事先知道哪些事件可以被消费？ Knative Eventing 在最新的 0.6 版本中提供 Registry 事件注册机制, 这样事件消费者就可以事先通过 Registry 获得哪些 Broker 中的事件类型可以被消费。

### Tekton
Tekton 是一个功能强大且灵活的Kubernetes原生CI/CD开源框架。通过抽象底层实现细节，用户可以跨多云平台和本地系统进行构建、测试和部署。具有组件化、声明式、可复用及云原生的特点。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573007354282-7dc4e2ed-4209-43db-a0cb-a225f3cfab41.png) 
- Tekton 是云原生的 - Cloud Native
   - 在Kubernetes上运行
   - 将Kubernetes集群作为第一选择
   - 基于容器进行构建
- Tekton 是解耦的 - Decoupled
   -  Pipeline 可用于部署到任何k8s集群
   - 组成 Pipeline 的Tasks 可以轻松地独立运行
   - 像 git repos这 样的 Resources 可以在运行期间轻松切换
- Tekton 是类型化的 - Typed
   - 类型化资源的概念意味着对于诸如的资源 Image，可以轻松地将实现切换（例如，使用 kaniko vs buildkit进行构建）
## Knative 的优势
Knative 一方面基于 Kubernetes 实现 Serverless 编排，另外一方面 Knative 还基于 Istio 实现服务的接入、服务路由的管理以及度发布等功能。Knative 是在已有的云原生基础之上构建的，有很好的社区基础。Knative 一经开源就得到了大家的追捧，其中的主要缘由有：
- Knative 的定位不是 PAAS 而是一个通用的 Serverless 编排框架，大家可以基于此框架构建自己的 Serverless PAAS
- Knative 对于 Serverless 架构的设计是标准先行。比如之前的 FAAS 解决方案每一个都有一套自己的事件标准，相互之间无法通用。而 Knative 首先提出了 CloudEvent 这一标准，然后基于此标准进行设计
- 完备的社区生态：Kubernetes、ServiceMesh 等社区都是 Knative 的支持者
- 跨平台：因为 Knative 是构建在 Kubernetes 之上的，而 Kubernetes 在不同的云厂商之间几乎可以提供通用的标准。所以 Knative 也就实现了跨平台的能力，没有供应商绑定的风险
- 完备的 Serverless 模型设计：和之前的 Serverless 解决方案相比 Knative 各方面的设计更加完备。比如：
  - 事件模型非常完备，有注册、订阅、以及外部事件系统的接入等等一整套的设计
  - 比如从源码到镜像的构建
  - 比如 Serving 可以做到按比例的灰度发布
  - 
## 总结
相信通过上面的介绍，大家对 Knative 有了初步的认识。Knative 使用 Tekton 提供云原生“从源代码到容器”的镜像构建能力，通过 Serving 部署容器并提供通用的服务模型，同时以 Eventing 提供事件全局订阅、传递和管理能力，实现事件驱动。这就是 Knative 呈现给我们的标准 Serverless 编排框架。