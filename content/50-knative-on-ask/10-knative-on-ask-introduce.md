---
title: "Knative On ASK 给您带来极致的 Serverless 体验"
date: 2020-5-11T15:26:15Z
weight: 10
---

CNCF 发布的[年度调查报告](https://www.cncf.io/wp-content/uploads/2020/03/CNCF_Survey_Report.pdf)显示 2019 年 Serverless 技术进一步获得了更大的认可。其中 41% 的受访者表示已经在使用 Serverless 另外 20% 的受访者表示计划在未来 12-18 个月内会采用 Serverless 技术。而在众多开源的 Serverless 项目中 Knative 是最受欢迎的一个。如下图所示 Knative 占据了 34% 的份额，遥遥领先于第二名 OpenFaaS，Knative 是自己搭建 Serverless 平台的首选。
![1588059613028-e149f4bf-719b-43c9-ab6a-8b244e727ec9.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/11431/1589187576974-482ca9b7-5ea2-4a91-a306-4552d96ec3dd.png) 
Knative 之所以这么受欢迎和容器的生态不无关系。和 FaaS 模式不同的是 Knative 不要求用户对应用做非常大的改造，只要用户的应用完成了容器化就能部署在 Knative 中。并且 Knative 在 Kubernetes 之上提供了更聚焦的应用模型，让用户无需为应用的升级、流量灰度花费精力，这一切都是自动完成的。

## 历史是胜利者的宣传
英国历史学家汤因比说过：“历史是胜利者的宣传”，那咱们就来看看 Serverless 的发展史告诉了我们什么。
从计算机的诞生到少数人开始使用计算机、从计算机在各个行业的普及再到全社会重度依赖计算机的这个过程自然的演化出了云计算、云原生和 serverless 的一系列理念。无论是云计算、云原生还是 Serverless，说到底这些都不是什么牛人自顶向下设计出来的，这些其实都是社会发展过程自然演化出来的，是一个自底向上的自然演化过程。既然是自底向上的自然演化那咱们先来捋一捋这个演化过程的核心脉络是什么。

在云计算出现之前，企业想要在互联网上对外提供服务需要先在 IDC 租赁物理机，然后再把应用部署在 IDC 的物理机上。而物理机的性能在过去十几年里始终保持着摩尔定律的速度在增长。这就导致单个应用根本无法充分利用整个物理机的资源。所以就需要有一种技术解决资源利用率的问题。简单的想如果一个应用不够就多部署几个。但在同一个物理机下多个应用混合部署会带来很多问题，比如：

- 端口冲突
  在同一台物理机上面每一个端口都是唯一的，不能被重复使用，如果一个应用占用了 80 端口那么所有其他的应用就都不能使用 80 端口了
- 资源隔离
  不同应用占用的资源不能做有效限制就容易相互影响，比如一个应用有 BUG 占光了内存就会导致其他应用出问题
- 系统依赖以及运维困难
  以 Nginx 为例，很多 web 应用都需要使用 Nginx 做反向代理。如果在 CentOS 上面安装 Nginx 一条简单的命令即可 yum install nginx 但如果不同的应用使用的 Nginx 版本不一样那就麻烦了。即便所有应用都可以使用一个版本的 Nginx ，那对于 Nginx 的维护来说也是一个棘手的问题。如果有一个 web 应用需要修改 Nginx 的配置，一旦配置错误就可能会导致所有应用都出现宕机的风险。
  
虚拟机的出现就很好的解决了上述问题，通过虚拟机技术可以在同一个物理机上虚拟出多个主机，每一个主机只部署一个应用，这样一台物理机不但可以部署多个应用，还能保证应用之间的独立性。
随着计算机技术和互联网的发展，企业和个人对主机的需求越来越多，对主机的需求也越来越复杂，比如网络隔离和安全、共享存储等。所以后来出现了专业的云服务商，虽然这些云服务商现在提供的产品都非常丰富，但最初都是从主机做起的，因为企业的第一诉求首先是有主机可以部署服务。至此云服务开始出现。

随着企业的发展，一家企业可能会维护很多应用。而每一个应用需要很多的发布、升级、回滚等操作，同时这些应用可能还需要在不同的地域分别部署一套。这就带来了很多运维负担，这些运维难题首当其冲的就是应用的运行环境。虚拟机技术虽然在宿主机之上隔离了不同的应用，但虚拟机还只是一个隔离的沙盒，对应用的部署并没有带来什么改变，而且虚拟机自身的开销也不小。所以后来出现了容器技术，容器技术通过内核级别的轻量级隔离能力不但具备与 VM 几乎同等的隔离体验，还带来了一个巨大的创建就是容器镜像。通过容器镜像可以非常容易的复制应用的运行环境。开发人员只需要把应用的依赖都做到镜像中，当镜像运行的时候直接使用自带的依赖提供服务。这就解决了应用发布、升级和回滚以及多地域部署等过程中的运行环境问题。

当人们开始大规模使用容器技术之后发现大大减轻了维护实例运行环境的负担，此时最大的问题就是应用多实例的协调以及多个应用之间的协调。比如：
- 单个应用的多个实例之间
  - 运行过程怎么检查实例服务运行是否正常
  - 当发现有一个实例不正常之后怎么访问入口中摘掉这个实例
  - 当实例所在的宿主机出现问题的时候怎么自动的在正常的机器上面创建一个新的实例补充进来
- 应用之间
  - 当前应用怎么暴露服务给其他应用
  - 当前应用底层实例切换怎样让调用方无感知
  - 多实例的负载均衡
  - 基于七层的 Ingress 策略管理等
  - 以及应用自身的快速部署和迁移

所以容器技术普及不久 Kubernetes 就出现了，Kubernetes 的出现就很好的解决了上述问题。

无论是早期的物理机模式，还是现在的 Kubernetes 模式应用开发者本身其实并不想管理什么底层资源，应用开发者只是想要把应用跑起来而已。在物理机模式中人们需要独占物理机，在 Kubernetes 模式中其实人们并不关心自己的业务进程是跑在哪个物理机上，事实上也无法提前预知。只要应用能够跑起来就好，至于是在哪儿运行的已其实不重要。物理机 -> 虚拟机 -> 容器 -> Kubernetes ，整个过程其实都是在简化应用使用 IaaS 资源的门槛。在这个演进过程可以发现一个清晰的脉络，**IaaS 和应用之间的耦合越来越低，基础平台只要做到在应用需要运行的时候给应用分配相应的 IaaS 资源即可，应用管理者只是 IaaS 的使用者，并不需要感知 IaaS 分配的细节**。
  
## Knative Serving
在介绍 Knative 之前咱们先通过一个 web 应用看一下在 Kubernetes 模式怎样做下做应用的流量接入和发布。如下图所示，在左侧是 Kubernetes 模式，右侧是 Knative 模式。

- 在 Kubernetes 模式下 
  - 用户需要自己管理 Ingress Controller
  - 要对外暴露服务需要维护 Ingress 和 Service 的关系
  - 发布时如果想要做灰度观察需要使用多个 Deployment 轮转才能完成升级
- 在 Knative 模式下
  - 用户只需要维护一个 Knative Service 资源即可

当然 Knative 并不能完全替代 Kubernetes ，Knative 是建立在 Kubernetes 能力之上的。Kubernetes 和 Knative 除了用户需要直接管理的资源不同，其实还有一个巨大的理念差异:
> Kubernetes 的作用是解耦 IaaS 和应用，降低 IaaS 资源分配的成本，Kubernetes 主要聚焦于 IaaS 资源的编排。而 Knative 是更偏向应用层，是以弹性为核心的应用编排。

![1599180788866-30945177-3ebd-4fdd-a71f-93e7f0304e3a.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/11431/1589187908595-ed2f78dd-e544-41d0-801d-82cca10197fb.png) 

Knative 是一款基于 Kubernetes 的 Serverless 编排引擎，其目标是制定云原生、跨平台的 Serverless 编排标准。Knative 通过整合容器构建、工作负载管理（动态扩缩）以及事件模型这三者来实现的这一 Serverless 标准。Serving 正是运行 Serverless 工作负载的核心模块。

![1609182391906-9581f075-7139-441d-86f3-eb3e358f6153.png](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/11431/1589187938665-ca3943a6-8451-44a9-81e1-40ea09172cc5.png) 

- 应用托管
  - Kubernetes 是面向 IaaS 管理的抽象，通过 Kubernetes 直接部署应用需要维护的资源比较多
  - 通过 Knative Service 一个资源就能定义应用的托管
- 流量管理
  - Knative 通过 Gateway 结果应用流量，然后可以对流量按百分比进行分割，这为弹性、灰度等基础能力做好了基础
- 灰度发布
  - 支持多版本管理，应用同时有多个版本在线提供服务很容易实现
  - 不同版本可以设置不同的流量百分比，对灰度发布等功能实现起来很容易
- 弹性
  - Knative 帮助应用节省成本的核心能力是弹性，在流量增加的时候自动扩容，流量下降的时候自动缩容
  - 每一个灰度的版本都有自己的弹性策略，并且弹性策略和分配到当前版本的流量是相关联的。Knative 会根据分配过来的流量多少进行扩容或者缩容的决策

Knative 更多介绍请移步[这里](https://knative.dev/)或[这里](http://knative-sample.com/)了解更多。

## 为什么是 ASK
ASK 的全称是 Serverless Kubernetes，是一种无服务器的 Kubernetes 集群，用户无需购买节点即可直接部署容器应用，无需对集群进行节点维护和容量规划，并且根据应用配置的 CPU 和内存资源量进行按需付费。ASK 集群提供完善的 Kubernetes 兼容能力，同时极大降低了 Kubernetes 使用门槛，让用户更专注于应用程序，而不是管理底层基础设施。

也就是说您无需提前准备 ECS 资源，即可直接创建一个 ASK 集群，然后就能在 ASK 集群中部署自己的服务了。关于 ASK 更详细的介绍参见[这里](https://help.aliyun.com/document_detail/86366.html?spm=a2c4g.11186623.6.935.6c874c07fK0OkT)。

前面分析 Serverless 历程的时候咱们总结出来的 Serverless 发展主脉络其实就是 **IaaS 和应用之间的耦合越来越低，基础平台只要做到在应用需要运行的时候给应用分配相应的 IaaS 资源即可，应用管理者只是 IaaS 的使用者，并不需要感知 IaaS 分配的细节**。ASK 就是这个随时分配 IaaS 资源的平台，Knative 负责感知应用的实时状态，在需要的时候自动从 ASK 中“申请”IaaS 资源(Pod)。Knative 和 ASK 的结合可以给您带来更极致的 Serverless 体验。


## 亮点介绍
### 基于 SLB 的 Gateway
Knative 社区默认支持 Istio、Gloo、Contour、Kourier 和 ambassador 等多种 Gateway 实现。在这些众多的实现中 Istio 当然是最流行的一个，因为 Istio 除了可以充当 Gateway 的角色还能做 ServiceMesh 服务使用。这些 Gateway 虽然功能完备但是作为 Serverless 服务的 Gateway 就有点儿违背初衷。首先需要有 Gateway 实例常驻运行，而为了保证高可用至少要有两个实例互为备份。其次这些 Gateway 的管控端也需要常驻运行，这些常驻实例的 IaaS 费用和运维都是业务需要支付的成本。
为了给用户提供极致的 Serverless 体验，我们通过阿里云 SLB 实现了 Knative  Gateway，所有需要的功能都具备而且是云产品级别的支撑。不需要常驻资源不但节省了您的 IaaS 成本还省去了很多运维负担。

### 低成本的保留实例
保留实例是 ASK Knative 独有的功能。社区的 Knative 默认在没有流量的时候可以缩容到零，但是缩容到零之后从零到一的冷启动问题很难解决。冷启动除了要解决 IaaS 资源的分配、Kubernetes 的调度、拉镜像等问题以外还涉及到应用的启动时长。应用启动时长从毫秒到分钟级别都有，这在通用的平台层面几乎无法控制。当然这些问题在所有的 serverless 产品中都存在。传统 FaaS 产品大多都是通过维护一个公共的 IaaS 池子运行不同的函数，为了保护这个池子不会被打满、和极低的冷启动时间，FaaS 产品的解法大多是对用户的函数做各种限制。比如：
- 处理请求的超时时间：如果超过这个时间没起来就认为失败
- 突增并发：默认所有函数都有一个并发上限，请求量超过这个上限就会被限流
- CPU 以及内存：不能超过 CPU 和内存的上限

ASK Knative 对这个问题的解法是通过低价格的保留实例来平衡成本和冷启动问题。阿里云 ECI 有很多规格，不同规格的计算能力不一样价格也不一样。如下所示对 2c4G 配置的计算型实例和突发性能型实例的价格对比。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/11431/1589188235521-c2fdbb00-e4d8-4c72-8b41-dcf3dd1813ab.png) 

通过上面的对比可知突发性能实例比计算型便宜 46%，可见如果在没有流量的时候使用突发性能实例提供服务不单单解决了冷启动的问题还能节省很多成本。

突发性能实例除了价格优势以外还有一个非常亮眼的功能就是CPU 积分。突发性能实例可以利用 CPU 积分应对突发性能需求。突发性能实例可以持续获得 CPU 积分，在性能无法满足负载要求时，可以通过消耗积累的 CPU 积分无缝提高计算性能，不会影响部署在实例上的环境和应用。通过 CPU 积分，您可以从整体业务角度分配计算资源，将业务平峰期剩余的计算能力无缝转移到高峰期使用(简单的理解就是油电混动啦☺️☺️)。突发性能实例的更多细节参见[这里](https://help.aliyun.com/document_detail/59977.html)。

所以 ASK Knative 的策略是在业务波谷时使用突发性能实例替换标准的计算型实例，当第一个请求来临时再无缝切换到标准的计算型实例。这样可以帮助您降低流量低谷的成本，并且在低谷时获得的 CPU 积分还能在业务高峰到来时消费掉，您支付的每一分钱都没有浪费。
使用突发性能实例作为保留实例只是默认策略，您可以指定自己期望的其他类型实例作为保留实例的规格。当然您也可以指定最小保留一个标准实例，从而关闭保留实例的功能。

## Demo 展示
[Serverless Kubernetes(ASK)](https://help.aliyun.com/document_detail/86366.html?spm=a2c4g.11186623.6.935.6c874c07RylkXG) 集群创建好以后可以通过以下钉钉群申请开通 Knative 功能。然后您就可以在 ASK 集群中直接使用 Knative 提供的能力了。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/11431/1589790086277-ca45986f-9e73-4a0c-b65b-a4b982783ae1.png) 

Knative 功能打开后会在 knative-serving namespace 中创建一个叫做 ingress-gateway 的 service，这是一个 loadbalance 类型的 service，会通过 ccm 自动创建一个 SLB 出来。如下所示 47.102.220.35 就是 SLB 的公网 IP，接下来就可以通过此公网 IP 访问 Knative 服务了。

```bash
# kubectl -n knative-serving get svc
NAME              TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
ingress-gateway   LoadBalancer   172.19.8.35   47.102.220.35   80:30695/TCP   26h
```

咱们后续的一系列操作就从一家咖啡店(cafe)开始谈起。假设这家咖啡店有两个品类：咖啡(coffee)和茶(tea)。咱们先部署 coffee 服务，然后再部署 tea 服务。同时还会包含版本升级、流量灰度、自定义 Ingress 以及自动弹性等特性的演示。

### 部署 coffee 服务
把如下内容保存到 coffee.yaml 文件中，然后通过 kubectl 部署到集群

```yaml
# cat coffee.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: coffee
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
          env:
            - name: TARGET
              value: "coffee"
```


执行 `kubectl apply -f coffee.yaml` 部署 coffee 服务，稍等一会儿应该可以看到 coffee 服务已经部署好了。

```bash
# kubectl get ksvc
NAME     URL                                 LATESTCREATED   LATESTREADY    READY   REASON
coffee   http://coffee.default.example.com   coffee-fh85b    coffee-fh85b   True
```

上面这个命令的执行结果中 `coffee.default.example.com` 就是 Knative 为每一个 ksvc 生成的唯一子域名。现在通过 curl 命令指定 host 和前面的 SLB 公网 IP 就能访问部署的这个服务了, 如下所示 `Hello coffee!` 就是 coffee 服务返回的内容。

```bash
# curl -H "Host: coffee.default.example.com" http://47.102.220.35
Hello coffee!
```
### 自动弹性
Autoscaler 是 Knative 的一等公民，这是 Knative 帮助用户节省成本的核心能力。Knative 默认的 KPA 弹性策略可以根据实时的流量请求自动调整 Pod 的数量。现在咱们就来感受一下 Knative 的弹性能力，先看一下当前的 pod 信息：

```bash
# kubectl get pod
NAME                                       READY   STATUS    RESTARTS   AGE
coffee-bwl9z-deployment-765d98766b-nvwmw   2/2     Running   0          42s
```

可以看到现在有 1 个 pod 在运行，接下来开始准备压测。在开始压测之前咱们先回顾一下 coffee 应用的配置，前面的 yaml 中有一个这样的配置, 如下所示，其中 autoscaling.knative.dev/target: "10" 意思就是每一个 Pod 的并发上限是 10 ，如果并发请求多余 10 就应该扩容新的 Pod 接受请求。关于 Knative Autoscaler 更详细的介绍请参见[这里](https://knative-sample.com/20-serving/10-autoscaler-kpa/)。

```yaml
# cat coffee-v2.yaml
... ...
      name: coffee-v2
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
... ...
```

好咱们现在开始压测一下试试。使用 hey 命令发起压测请求， hey 命令下载链接：
- macOS : https://knative-samples.oss-cn-shanghai.aliyuncs.com/hey/darwin/hey
- Linux : https://knative-samples.oss-cn-shanghai.aliyuncs.com/hey/linux/hey
- Windows： https://knative-samples.oss-cn-shanghai.aliyuncs.com/hey/windows/hey.exe

```bash
hey -z 30s -c 90 --host "coffee.default.example.com" "http://47.100.6.39/?sleep=100"
```

上面这条命令的含义：
- -z 30s 表示连续压测 30 秒
- -c 90 表示使用 90 个并发请求发起压测
- --host "coffee.default.example.com" 表示绑定 host
- "http://47.100.6.39/?sleep=100" 就是请求 url，其中 sleep=100 在测试镜像中表示 sleep 100 毫秒，模拟真实的线上服务

执行上述命令进行压测，然后观察 Pod 数量的变化情况。可以使用 watch -n 1 'kubectl  get pod' 这条命令查看 Pod 监测 Pod 数量。如下所示做半边是 Pod 数量的变化，右半边是压测命令执行的过程。可以通过这个 gif 图观察一下压测的过程中 pod 的变化情况。当压力上来之后 Knative 就会自动扩容，所以 Pod 数量就会变多。当压测结束之后 Knative 监测到流量变少就会自动缩容，这是一个完全自动化的扩容和缩容的过程。

![out.gif](https://intranetproxy.alipay.com/skylark/lark/0/2020/gif/11431/1589188997877-f958eb8f-de13-4b1b-9647-5204aa50d707.gif) 

### 保留实例
前面亮点一节中介绍了 ASK Knative 会使用保留实例解决冷启动和成本问题。接下来咱们就看一下保留实例和标准实例的切换过程。
前面的压测结束以后多过一会儿，再使用 kubectl get pod 查看一下 Pod 的数量可能会发现只有一个 Pod 了，并且 Pod  名字是 xxx-reserve-xx  这种，reserve 的含义就是保留实例。这时候其实已经在使用保留实例提供服务了。当线上长时间没有请求之后 Knative 就会自动扩容保留实例，并且把标准实例缩容到零，从而达到节约成本的目的。

```bash
# kubectl get pod
NAME                                               READY   STATUS    RESTARTS   AGE
coffee-bwl9z-deployment-reserve-85fd89b567-vpwqc   2/2     Running   0          5m24s
```
如果此时有流量进来会发生什么呢？咱们就来验证一下。通过下面的 gif 可以看出此时如果有流量进来就会自动扩容标准实例，待标准实例 ready 之后保留实例就会被缩容。
![2020-05-08_192527.gif](https://intranetproxy.alipay.com/skylark/lark/0/2020/gif/11431/1589188877384-aaf859c1-ff7d-4ed5-91e5-6628ecf5f66d.gif) 

保留实例默认会使用 [ecs.t5-lc1m2.small(1c2g)](https://help.aliyun.com/document_detail/59977.html#section-mq2-x7y-0jl) 这个规格。当然有些应用默认启动的时候就要分配好内存(比如 JVM)，假设有一个应用需要 4G 的内存，则可能需要把 [ecs.t5-c1m2.large(2c4g)](https://help.aliyun.com/document_detail/59977.html#section-mq2-x7y-0jl) 作为保留实例的规格。所以我们也提供了用户指定保留实例规格的方法，用户在提交 knative Service 时可以通过 Annotation 的方式指定保留实例的规格, 比如 `knative.aliyun.com/reserve-instance-eci-use-specs: ecs.t5-lc2m1.nano` 这个配置的意思是使用 `ecs.t5-lc2m1.nano` 作为保留实例规格。 把下面内容保存到 `coffee-set-reserve.yaml` 文件：

```yaml
# cat coffee-set-reserve.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: coffee
spec:
  template:
    metadata:
      annotations:
        knative.aliyun.com/reserve-instance-eci-use-specs: "ecs.t5-c1m2.large"
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
          env:
            - name: TARGET
              value: "coffee-set-reserve"
```
执行 `kubectl apply -f coffee-set-reserve.yaml` 提交到 Kubernetes 集群。稍微等待一会儿，待新版本缩容到保留实例之后查看一下 Pod 列表：

```bash
# kubectl get pod
NAME                                               READY   STATUS    RESTARTS   AGE
coffee-vvfd8-deployment-reserve-845f79b494-lkmh9   2/2     Running   0          2m37s
```
查看 `set-reserve` 的保留实例规格， 可以看到已经设置成 2c4g 的 `ecs.t5-c1m2.large` 规格

```yaml
# kubectl get pod coffee-vvfd8-deployment-reserve-845f79b494-lkmh9 -oyaml |head -20
apiVersion: v1
kind: Pod
metadata:
  annotations:
    ... ...
    k8s.aliyun.com/eci-instance-cpu: "2.000"
    k8s.aliyun.com/eci-instance-mem: "4.00"
    k8s.aliyun.com/eci-instance-spec: ecs.t5-c1m2.large
    ... ...
```

### 升级 coffee 服务
在升级之前咱们先看一下现在的 Pod 实例：
```bash
# kubectl get pod
NAME                                               READY   STATUS    RESTARTS   AGE
coffee-fh85b-deployment-8589564f7b-4lsnf           1/2     Running   0          26s
```
现在咱们来对 coffee 服务做一次升级。把下面的内容保存到 `coffee-v1.yaml` 文件中：

```yaml
# cat coffee-v1.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: coffee
spec:
  template:
    metadata:
      name: coffee-v1
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
          env:
            - name: TARGET
              value: "coffee-v1"
```
当前部署的这个版本中添加了两点：
- 给当前部署的 revision 设置了一个名字 coffee-v1 (如果不设置就自动生成)
- • 环境变量中设置了 v1 字样，这样可以通过 http 返回的内容中判断当前提供服务的是 v1 版本
执行  `kubectl apply -f coffee-v1.yaml` 命令部署 v1 版本。部署完以后继续使用 `curl -H "Host: coffee.default.example.com" http://47.102.220.35` 进行验证。

过几秒钟可以发现返回的内容是 `Hello coffee-v1!` ,在这个过程中服务没有中断，也不需要手动做任何切换，修改完以后直接提交即可自动完成新版本和老版本实例的切换。

```bash
# curl -H "Host: coffee.default.example.com" http://47.102.220.35
Hello coffee-v1!
```
现在咱们再来看一下 pod 实例的状态, 可以见 Pod 实例发生了切换。老版本的 Pod 自动被新版本替换掉了。

```bash
# kubectl get pod
NAME                                    READY   STATUS    RESTARTS   AGE
coffee-v1-deployment-5c5b59b484-z48gm   2/2     Running   0          54s
```

还有更多复杂功能的 Demo 演示，请移步[这里](https://knative-sample.com//50-knative-on-ask/20-introduce-demo/)

## 总结
Knative 是 Kubernetes 生态最流行的 Serverless 编排框架。社区原生的 Knative 需要常驻的 Controller 和常驻的网关才能提供服务。这些常驻实例除了需要支付 IaaS 成本以外还带来了很多运维负担，给应用的 Serverless 化带来了一定的难度。所以我们在 ASK 中完全托管了 Knative Serving。开箱即用，您不需要为这些常驻实例付出任何成本。除了通过 SLB 云产品提供 Gateway 的能力以外我们还提供了基于突发性能型实例的保留规格功能，这样可以让您的服务在流量波谷时期大大的减少 IaaS 开支，并且流量波谷时期积攒的 CPU 积分可以在流量高峰时期消费，您支付的每一分钱都没有浪费。

更多 Knative 的资料请参见[这里](https://knative.dev/)或者[这里](https://knative-sample.com/) 
## 参考资料
- https://help.aliyun.com/document_detail/59977.html
- https://help.aliyun.com/document_detail/59977.html#section-i6c-2kn-y5s
- https://help.aliyun.com/document_detail/90634.html?spm=a2c4g.11186623.2.30.7f7f61a1qMhXB1#section-cds-zh0-eh7
