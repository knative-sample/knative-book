---
title: "Demo 演示"
date: 2020-5-11T15:26:15Z
draft: false
weight: 20
---

阿里云 [Serverless Kubernetes(ASK)](https://help.aliyun.com/document_detail/86366.html?spm=a2c4g.11186623.6.935.6c874c07RylkXG) 集群创建好以后使用钉钉扫描下面这个二维码，到【Serverless Kubernetes/ECI用户群】群申请开通 Knative 。然后您就可以在 ASK 集群中直接使用 Knative 提供的能力了。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2020/png/11431/1589790086277-ca45986f-9e73-4a0c-b65b-a4b982783ae1.png) 


Knative 功能打开后会在 knative-serving namespace 中创建一个叫做 ingress-gateway 的 service，这是一个 loadbalance 类型的 service，会通过 ccm 自动创建一个 SLB 出来。如下所示 47.100.6.39 就是 SLB 的公网 IP，接下来就可以通过此公网 IP 访问 Knative 服务了。

```bash
# kubectl -n knative-serving get svc
NAME              TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
ingress-gateway   LoadBalancer   172.19.8.35   47.100.6.39   80:30695/TCP   26h
```

咱们后续的一系列操作就从一家咖啡店(cafe)开始谈起。假设这家咖啡店有两个品类：咖啡(coffee)和茶(tea)。咱们先部署 coffee 服务，然后再部署 tea 服务。同时还会包含版本升级、流量灰度、自定义 Ingress 以及自动弹性等特性的演示。

## 部署 coffee 服务
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
# curl -H "Host: coffee.default.example.com" http://47.100.6.39
Hello coffee!
```
## 自动弹性
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

## 保留实例
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

## 升级 coffee 服务
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
执行  `kubectl apply -f coffee-v1.yaml` 命令部署 v1 版本。部署完以后继续使用 `curl -H "Host: coffee.default.example.com" http://47.100.6.39` 进行验证。

过几秒钟可以发现返回的内容是 `Hello coffee-v1!` ,在这个过程中服务没有中断，也不需要手动做任何切换，修改完以后直接提交即可自动完成新版本和老版本实例的切换。

```bash
# curl -H "Host: coffee.default.example.com" http://47.100.6.39
Hello coffee-v1!
```
现在咱们再来看一下 pod 实例的状态, 可以见 Pod 实例发生了切换。老版本的 Pod 自动被新版本替换掉了。

```bash
# kubectl get pod
NAME                                    READY   STATUS    RESTARTS   AGE
coffee-v1-deployment-5c5b59b484-z48gm   2/2     Running   0          54s
```

## 流量调配
前面咱们已经配置好了 v1 版本，现在咱们再部署一个 v2 版本，并设置 30% 的流量到 v2, 70% 的流量到 v1 。把下面的内容保存到 `coffee-v2.yaml`
```yaml
# cat coffee-v2.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: coffee
spec:
  template:
    metadata:
      name: coffee-v2
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
          env:
            - name: TARGET
              value: "coffee-v2"

  traffic:
  - tag: v2
    revisionName: coffee-v2
    percent: 30

  - tag: v1
    revisionName: coffee-v1
    percent: 70
```

执行 `kubectl apply -f coffee-v2.yaml` ，然后继续通过 curl 命令观察返回的内容。因为需要观察多次，所以这里对 curl 命令做了一些调整  `while true; do curl -H "Host: coffee.default.example.com" http://47.100.6.39; done`    会看到有大概 30% 的返回是 v2，有大概 70% 的返回是 v1

```bash
# while true; do curl -H "Host: coffee.default.example.com" http://47.100.6.39; done
... ...
Hello coffee-v1!
Hello coffee-v2!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v2!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v2!
Hello coffee-v1!
Hello coffee-v2!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v2!
Hello coffee-v2!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v2!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v1!
... ...
```
更多 Knative 流量分配方法请参见[这里](https://knative-sample.com/20-serving/30-traffic-and-version/)。

## 多版本的自动弹性
好，现在保持两个版本的状态，咱们来看看现在的 pod 实例情况如下，其中 coffee-v1 开头的是 v1 版本，coffee-v2 开头的是 v2 版本。
```bash
# kubectl get pod
NAME                                    READY   STATUS    RESTARTS   AGE
coffee-v1-deployment-5c5b59b484-mcx56   2/2     Running   0          42s
coffee-v2-deployment-756d9c85b8-5g2p5   2/2     Running   0          66s
```


现在有两个 Pod，每一个版本有一个 pod 在运行。那么如果此时来一波高并发请求会发生什么呢？在开始压测之前咱们先回顾一下 coffee 应用的配置，前面的 yaml 中有一个这样的配置, 如下所示，其中 `autoscaling.knative.dev/target: "10"` 意思就是每一个 Pod 的并发上限是 10 ，如果并发请求多余 10 就应该扩容新的 Pod 接受请求。更多 Knative Autoscaler 配置方法请参见[这里](https://knative-sample.com/20-serving/10-autoscaler-kpa/)。
```bash
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
好咱们现在开始压测一下试试。使用 hey 命令发起压测请求：
```bash
hey -z 30s -c 100 --host "coffee.default.example.com" "http://47.100.6.39/?sleep=100"
```
上面这条命令的含义：
- -z 30s 表示连续压测 30 秒
- -c 100 表示使用 100 个并发请求发起压测
- --host "coffee.default.example.com" 表示绑定 host 
- "http://47.100.6.39/?sleep=100" 就是请求 url，其中 sleep=100 在测试镜像中表示 sleep 100 毫秒，模拟真实的线上服务

压测命令结束后观察一下 Pod 的运行情况(在压测过程中也可以观察，只是需要打开两个命令行窗口)，你可以会看到下面这样的 Pod 版本分布。可以看到 Knative 驱动的 coffee 应用根据实时的流量情况自动对 Pod 数量做了扩容，保证应用实例数始终是符合实时的流量需求。v1 pod 数量对 v2  pod 数量的比例和 v1 对 v2 的流量比例基本一致。

```bash
# kubectl get pod
NAME                                    READY   STATUS    RESTARTS   AGE
coffee-v1-deployment-5c5b59b484-gsccg   2/2     Running   0          46s
coffee-v1-deployment-5c5b59b484-mbc5n   2/2     Running   0          50s
coffee-v1-deployment-5c5b59b484-mcx56   2/2     Running   0          2m4s
coffee-v1-deployment-5c5b59b484-n85xf   2/2     Running   0          46s
coffee-v1-deployment-5c5b59b484-tvwgv   2/2     Running   0          48s
coffee-v1-deployment-5c5b59b484-xtd7k   2/2     Running   0          50s
coffee-v1-deployment-5c5b59b484-zmpnw   2/2     Running   0          46s
coffee-v2-deployment-756d9c85b8-5g2p5   2/2     Running   0          2m28s
coffee-v2-deployment-756d9c85b8-p9mhm   2/2     Running   0          48s
coffee-v2-deployment-756d9c85b8-vnw8s   2/2     Running   0          48s
```
过一会儿再获取一下 pod 列表可能会看到下面这样的信息，可以看到当前 v1 和 v2 版本都已经缩容到保留实例了。

```bash
# kubectl get pod
NAME                                            READY   STATUS    RESTARTS   AGE
coffee-v1-deployment-reserve-7759668c89-rsxgx   2/2     Running   0          107s
coffee-v2-deployment-reserve-5d68bd9565-mf4gq   2/2     Running   0          107s
```
## ingress 功能
咖啡店除了有 coffee 一般还会有 tea，所以咱们再来部署一个 tea 服务。 coffee 和 tea 都通过 cafe.knative-ask.cs.aliyun.com 域名对外提供服务，其中 `/coffee`  path 对应 coffee 服务，`/tea` path 对应 tea 服务。
部署 tea 服务，把下面内容保存到 `tea-with-ingress.yaml` 中：

```yaml
# cat tea-with-ingress.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: tea
  annotations:
    knative.aliyun.com/serving-ingress: cafe.knative-ask.cs.aliyun.com/tea
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
              value: "tea"
```

其中 `knative.aliyun.com/serving-ingress: cafe.knative-ask.cs.aliyun.com/tea` 部分就是配置 Ingress 的域名和 path。现在 `kubectl apply -f tea-with-ingress.yaml`  就把 tea 服务部署好了。稍等片刻，tea pod  Running 之后可以访问 tea 服务了。
```bash
# curl -H "Host: cafe.knative-ask.cs.aliyun.com" http://47.100.6.39/tea
Hello tea!
```
coffee 服务还没有设置 Ingress，咱们给 coffee 也添加一个 domain 和 path，把下面的内容保存到 `coffee-v3-ingress.yaml` 文件中

```yaml
# cat coffee-v3-ingress.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: coffee
  annotations:
    knative.aliyun.com/serving-ingress: cafe.knative-ask.cs.aliyun.com/coffee
spec:
  template:
    metadata:
      name: coffee-v3
      annotations:
        knative.aliyun.com/reserve-instance-eci-use-specs: "ecs.t5-c1m2.large"
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
          env:
            - name: TARGET
              value: "coffee-v3"

  traffic:
  - tag: v3
    revisionName: coffee-v3
    percent: 10
  - tag: v2
    revisionName: coffee-v2
    percent: 20

  - tag: v1
    revisionName: coffee-v1
    percent: 70
```
`kubectl apply -f coffee-v3-ingress.yaml` 部署到 K8s 中，然后过几秒在通过 curl 指定 cafe.knative-ask.cs.aliyun.com  host 和 /coffee path 访问可以看到请求已经被转发到 coffee 服务，并且按照 Knative Service traffic 中的配置进行了流量分配。
```bash
# while true; do curl -H "Host: cafe.knative-ask.cs.aliyun.com" http:/47.100.6.39/coffee; done
Hello coffee-v1!
Hello coffee-v2!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v2!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v1!
Hello coffee-v3!
```

## 总结
Knative 是 Kubernetes 生态最流行的 Serverless 编排框架。社区原生的 Knative 需要常驻的 Controller 和常驻的网关才能提供服务。这些常驻实例除了支付 IaaS 成本还带来了很多运维负担，给 Serverless 服务带来了一定的难度。所以我们在 ASK 中完全托管了 Knative Serving，开箱即用您只需要使用 Serverless 的功能，而不需要为这些常驻实例付出任何成本。除了通过 SLB 云产品提供 Gateway 的能力以外我们还提供了基于突发性能型实例的保留规格功能，这样可以让您的服务在流量波谷时期大大的减少 IaaS 开支，并且流量波谷时期积攒的 CPU 积分可以在流量高峰时期消费，您支付的每一分钱都没有浪费。除了这些特性以外本文还展示了 Knative 的下面几个特性：

- 应用部署和升级功能
- 应用多版本管理以及多个版本基于流量比例分别提供服务的能力
- 基于实时流量按需分配实例的自动弹性能力
- Ingress 功能

## 参考资料
- https://help.aliyun.com/document_detail/59977.html
- https://help.aliyun.com/document_detail/59977.html#section-i6c-2kn-y5s
- https://help.aliyun.com/document_detail/90634.html?spm=a2c4g.11186623.2.30.7f7f61a1qMhXB1#section-cds-zh0-eh7
