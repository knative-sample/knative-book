---
title: "Serving Hello World"
date: 2020-5-10T15:26:15Z
draft: false
weight: 40
---

通过前面两章的学习你已经掌握了很多 Knative 的理论知识，基于这些知识你应该对 Knative 是谁、它来自哪里以及它要做什么有了一定的认识。可是即便如此你可能还是会有一种犹抱琵琶半遮面，看不清真容的感觉，这就好比红娘拿姑娘的 100 张生活照给你看也不如你亲自去见一面。按常理出牌，一般到这个阶段就该 Hello World 出场了。本篇文章就通过一个 Hello World 和 Knative 来一个“约会”，让你一睹 Knative 这位白富美的真容。

Serverless 一个核心思想就是按需分配，那么 Knative 是如何实现按需分配的呢？另外在前面的文章中你已经了解到 Knative Serving 在没有流量的时候是可以把 Pod 缩容到零的。接下来就通过一些例子体验一下 Knative 缩容到零和按需自动扩缩容的能力。
### 部署 helloworld-go 示例
Knative 官方给出了好几种语言的 [Helloworld 示例](https://github.com/knative/docs/tree/master/docs/serving/samples/hello-world)，这些不同的语言其实只是编译镜像的 Dockerfile 有所不同，做好镜像之后的使用方式没什么差异。本例以 go 的 Hello World 为例进行演示。官方给出的例子都是源码，需要编译长镜像才能使用。为了你验证方便我已经提前编译好了一份镜像  registry.cn-hangzhou.aliyuncs.com/knative-sample/simple-app:07 , 你可以直接使用。

首先编写一个 Knative Service 的 yaml 文件 `helloworld-go.yaml` , 内容如下：
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld-go
spec:
  template:
    metadata:
      labels:
        app: helloworld-go
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
          ports:
            - name: http1
              containerPort: 8080
          env:
            - name: TARGET
              value: "World"
```
注意其中 `autoscaling.knative.dev/target: "10"` 这个 Annotation 是设置每一个 Pod 的可处理并发请求数 10 ，Knative KPA 自动伸缩的时候会根据当前总请求的并发数和 `autoscaling.knative.dev/target` 自动调整 Pod 的数量，从而达到自动扩缩的目的。更多的策略信息我会在后续的文章中一一介绍。
现在使用 kubectl 命令把 yaml 提交到 Kubernetes 中:

- 部署 helloworld-go
```bash
└─# kubectl apply -f helloworld-go.yaml
service.serving.knative.dev/helloworld-go created
```

- 查看 helloworld-go pod
```bash
└─# kubectl get pod
NAME                                                              READY   STATUS      RESTARTS   AGE
helloworld-go-7n4dm-deployment-65cb6d9bf-9njqx                    2/2     Running     0          8s
```
到此 helloworld-go 已经运行起来了，接下来访问一下  helloworld-go 这个服务吧。
### 访问 helloworld-go 示例
在访问 helloworld-go 之前我要先来介绍一下在 Knative 模型中流量是怎么进来的。Knative Service 和 Kubernetes 原生的 Deployment 不一样，Knative 不会创建 Loadbalance 的 Service，也不能创建 NodePort 类型的 Service，所以不能通过 SLB 或者 NodePort 访问。只能通过 ClusterIP 访问。而 ClusterIP 是不能直接对外暴露的，所以必须经过 Gateway 才能把用户的流量接入进来。本例就是使用 Istio 的 Gateway 承接 Knative 的南北流量(进和出)。如下图所示是 Knative 模型中流量的转发路径。用户发起的请求首先会打到 Gateway 上面，然后 Istio 通过 VirtualService 再把请求转发到具体的  Revision 上面。当然用户的流量还会经过 Knative 的 queue 容器才能真正转发到业务容器，关于这方面的细节我在后续的文章再进行详细的介绍。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1559533922410-7a51e5be-fd95-4c7c-93b9-888599c01368.png) 
所以想要访问 Knative 的服务首先要获取 Gateway 的 IP 地址，可以通过如下方式获取 Gateway 的 IP：

```bash
└─# kubectl get svc istio-ingressgateway --namespace istio-system --output jsonpath="{.status.loadBalancer.ingress[*].ip}"
39.106.232.122
```

前面也介绍了 Gateway 是通过 VirtualService 来进行流量转发的，这就要求访问者要知道目标服务的名字才行(域名)，所以要先获取 helloworld-go 的域名, 注意下面这条命令中的 `${SVC_NAME}` 需要替换成 `helloworld-go` ，这个名字必须要和 Knative Service 的名字一致，因为每一个 Service 都有一个唯一的名字。

```bash
└─# kubectl get route ${SVC_NAME} --output jsonpath="{.status.domain}"
helloworld-go.default.knative.kuberun.com
```

至此你已经拿到 IP 地址和 Hostname，可以通过 curl 直接发起请求：

```bash
└─# curl -H "Host: helloworld-go.default.knative.kuberun.com" "http://39.106.232.122"
Hello World!
```

### 缩容到零
刚刚部署完 Service 的时候 Knative 默认会创建出一个 Pod 提供服务，如果你超过 90 秒没有访问 helloworld-go 这个服务那么这个 Pod 就会自动删除，此时就是缩容到零了。现在看一下 Pod 情况, 你可能会发现没有 Pod

```bash
└─# kubectl get pod -o wide
No resources found.
```

```bash
└─# time curl -H "Host: helloworld-go.default.knative.kuberun.com" "http://39.106.232.122"
Hello World!

real	0m2.775s
user	0m0.007s
sys	0m0.007s
```
注意结果中，这面这一段：
```bash
real	0m2.775s
user	0m0.007s
sys	0m0.007s
```
`real	0m2.775s`  意思意思是 `curl` 请求执行一共消耗了 `2.775s` , 也就是说 Knative 从零到 1 扩容 + 启动容器再到服务响应请求总共消耗了 `2.775s` （我之前的测试导致在 Node  上面缓存了镜像，所以没有拉镜像的时间）。可以看出来这个速度还是很快的。

再看一下 pod 数量， 你会发现此时 Pod 自动扩容出来了。并且 Pod 数量为零时发起的请求并没有拒绝链接。
```bash
└─# kubectl get pod
NAME                                              READY   STATUS    RESTARTS   AGE
helloworld-go-p9w6c-deployment-5dfdb6bccb-gjfxj   2/2     Running   0          31s
```
### 按需分配，自动扩缩
接下来再测试一下 Knative 按需扩容的功能。使用社区提供的 hey 进行测试。hey 有 Windows、Linux 和 Mac 的二进制可以在[这里下载](https://github.com/rakyll/hey/releases)。
使用这个命令测试之前需要在本机进行 Host 绑定，对于 helloworld-go 来说要把 helloworld-go 的域名绑定到 Istio Gateway 的 IP 上，/etc/hosts 添加如下配置
```
39.106.232.122  helloworld-go.default.knative.kuberun.com
```

如下所示 这条命令的意思是:
- `-z 30s` 持续测试  30s
- `-c 50` 保持每秒 50 个请求

测试结果如下：

```bash
└─# hey -z 30s -c 50 "http://helloworld-go.default.knative.kuberun.com/" && kubectl get pods

Summary:
  Total:	30.0388 secs
  Slowest:	2.6178 secs
  Fastest:	0.0276 secs
  Average:	0.0434 secs
  Requests/sec:	1152.3116

  Total data:	449982 bytes
  Size/request:	13 bytes

Response time histogram:
  0.028 [1]	|
  0.287 [34559]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.546 [4]	|
  0.805 [0]	|
  1.064 [0]	|
  1.323 [0]	|
  1.582 [0]	|
  1.841 [0]	|
  2.100 [0]	|
  2.359 [0]	|
  2.618 [50]	|


Latency distribution:
  10% in 0.0320 secs
  25% in 0.0346 secs
  50% in 0.0380 secs
  75% in 0.0424 secs
  90% in 0.0480 secs
  95% in 0.0532 secs
  99% in 0.0775 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0001 secs, 0.0276 secs, 2.6178 secs
  DNS-lookup:	0.0000 secs, 0.0000 secs, 0.0185 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0023 secs
  resp wait:	0.0431 secs, 0.0273 secs, 2.5646 secs
  resp read:	0.0001 secs, 0.0000 secs, 0.0195 secs

Status code distribution:
  [200]	34614 responses

NAME                                                              READY   STATUS             RESTARTS   AGE
helloworld-go-7n4dm-deployment-65cb6d9bf-8x6b5                    2/2     Running            0          29s
helloworld-go-7n4dm-deployment-65cb6d9bf-dvrm5                    2/2     Running            0          27s
helloworld-go-7n4dm-deployment-65cb6d9bf-f7db8                    2/2     Running            0          30s
helloworld-go-7n4dm-deployment-65cb6d9bf-fsn2z                    2/2     Running            0          29s
helloworld-go-7n4dm-deployment-65cb6d9bf-rmrh2                    2/2     Running            0          29s
```

回想一下刚才 helloworld-go.yaml 文件配置，已经设置了 `autoscaling.knative.dev/target: "10"` 这个 Annotation。这表示每一个 Pod 能够接受并发 10 个请求，而刚才并发请求数设置的是 50 所以理论上应该会创建出来 5 个 Pod?，

上面结果中最后一部分,是 `kubectl get pods` 的结果，如下所示：
```
NAME                                                              READY   STATUS             RESTARTS   AGE
helloworld-go-7n4dm-deployment-65cb6d9bf-8x6b5                    2/2     Running            0          29s
helloworld-go-7n4dm-deployment-65cb6d9bf-dvrm5                    2/2     Running            0          27s
helloworld-go-7n4dm-deployment-65cb6d9bf-f7db8                    2/2     Running            0          30s
helloworld-go-7n4dm-deployment-65cb6d9bf-fsn2z                    2/2     Running            0          29s
helloworld-go-7n4dm-deployment-65cb6d9bf-rmrh2                    2/2     Running            0          29s
```

可以看到此时 Knative 自动扩容出来了 5 个 Pod 处理请求。

## 总结
至此你已经完成了和 Knative Serving 的首次约会，也看到了这位白富美的真容。通过本篇文章你应该掌握
以下几点：
- 理解 Knative 从零到一的含义，并且能够基于 helloworld-go 例子演示这个过程
- 理解 Knative 按需扩缩容的含义，并且能够基于 autoscale-go 例子演示这个过程
- 理解 Knative KPA 按需扩容的原理

## 参考文档
- 示例代码： https://github.com/knative-sample/helloworld-go/tree/b1.0 
