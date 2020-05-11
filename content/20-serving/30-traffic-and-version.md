---
title: "流量灰度和版本管理"
date: 2020-5-10T15:26:15Z
draft: false
weight: 30
description: "calling built-in Shortcodes into your content files."
---


本篇主要介绍 Knative Serving 的流量灰度，通过一个 rest-api 的例子演示如何创建多个 Revision、并在不同的 Revision 之间按照流量比例灰度。

## 部署 rest-api v1
咱们先部署第一个版本。
- 代码 
测试之前我们需要写一段  rest-api 的代码，并且还要能够区分不同的版本。下面我基于官方的[例子](https://github.com/knative/docs/blob/master/docs/serving/samples/rest-api-go/stock.go)进行了修改，为了使用方便去掉了 `github.com/gorilla/mux` 依赖，直接使用 Golang 系统包  `net/http` 替代。这段代码可以通过 RESOURCE 环境变量来区分不同的版本。

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"net/url"
	"os"

	"flag"
)

var resource string

func main() {
	flag.Parse()
	//router := mux.NewRouter().StrictSlash(true)

	resource = os.Getenv("RESOURCE")
	if resource == "" {
		resource = "NOT SPECIFIED"
	}

	root := "/" + resource
	path := root + "/{stockId}"

	http.HandleFunc("/", Index)
	http.HandleFunc(root, StockIndex)
	http.HandleFunc(path, StockPrice)

	if err := http.ListenAndServe(fmt.Sprintf(":%s", "8080"), nil); err != nil {
		log.Fatalf("ListenAndServe error:%s ", err.Error())
	}
}

func Index(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Welcome to the %s app! \n", resource)
}

func StockIndex(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "%s ticker not found!, require /%s/{ticker}\n", resource, resource)
}

func StockPrice(w http.ResponseWriter, r *http.Request) {
	stockId := r.URL.Query().Get("stockId")

	url := url.URL{
		Scheme: "https",
		Host:   "api.iextrading.com",
		Path:   "/1.0/stock/" + stockId + "/price",
	}

	log.Print(url)

	resp, err := http.Get(url.String())
	if err != nil {
		fmt.Fprintf(w, "%s not found for ticker : %s \n", resource, stockId)
		return
	}

	defer resp.Body.Close()

	body, err := ioutil.ReadAll(resp.Body)

	fmt.Fprintf(w, "%s price for ticker %s is %s\n", resource, stockId, string(body))
}
```
- Dockerfile
创建一个叫做 Dockerfile 的文件，把下面这些内容复制到文件中。执行 `docker build --tag registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1 --file ./Dockerfile .`  命令即可完成镜像的编译。
你在测试的时候请把 `registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1` 换成你自己的镜像仓库地址。
编译好镜像以后执行 `docker push registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1` 把镜像推送到镜像仓库。

```dockerfile
FROM registry.cn-hangzhou.aliyuncs.com/knative-sample/golang:1.12 as builder

WORKDIR /go/src/github.com/knative-sample/rest-api-go
COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -v -o rest-api-go
FROM registry.cn-hangzhou.aliyuncs.com/knative-sample/alpine-sh:3.9
COPY --from=builder /go/src/github.com/knative-sample/rest-api-go/rest-api-go /rest-api-go

CMD ["/rest-api-go"]
```

 - Service 配置
 镜像已经有了，我们开始部署 Knative Service。把下面的内容保存到 revision-v1.yaml 中，然后执行 `kubectl apply -f revision-v1.yaml ` 即可完成 Knative Service 的部署。
 
 ```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: traffic-example
  namespace: default
spec:
  template:
    metadata:
      name: traffic-example-v1
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1
        env:
          - name: RESOURCE
            value: v1
        readinessProbe:
          httpGet:
            path: / 
 ```
 
首次安装会创建出一个叫做 `traffic-example-v1` 的 Revision，并且是把 100% 的流量都打到 `traffic-example-v1` 上。

**验证 Serving 的各个资源**
如下图所示，我们先回顾一下 Serving 涉及到的各种资源。接下来我们分别看一下刚才部署的 `revision-v1.yaml` 各个资源配置。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1563317462047-cfe83f6a-d47b-4a51-9c23-b9edc0d3b0fb.png) 

- Knative Service 
```bash
kubectl get ksvc traffic-example --output yaml
```
- Knative Configuration
```bash
kubectl get configuration -l \
"serving.knative.dev/service=traffic-example" --output yaml
```
- Knative Revision
```bash
kubectl get revision -l \
"serving.knative.dev/service=traffic-example" --output yaml
```
- Knative Route
```bash
kubectl get route -l \
"serving.knative.dev/service=traffic-example" --output yaml
```
**访问 rest-api 服务**
我们部署的 Service 名称是： traffic-example。访问这个 Service 需要获取 Istio Gateway 的 IP，然后使用 traffic-example.default.knative.kuberun.com 这个 Domain 绑定 Host 的方式发起 curl 请求。为了方便测试我写成了一个脚本。创建一个 run-test.sh 文件，把下面这些内容复制到文件内，然后赋予文件可执行权限。执行执行此脚本就能得到测试结果。
```bash
#!/bin/bash
#****************************************************************#
# Create Date: 2019-11-06 14:38
#********************************* ******************************#

SVC_NAME="traffic-example"
export INGRESSGATEWAY=istio-ingressgateway
export GATEWAY_IP=`kubectl get svc $INGRESSGATEWAY --namespace istio-system --output jsonpath="{.status.loadBalancer.ingress[*]['ip']}"`
export DOMAIN_NAME=`kubectl get route ${SVC_NAME} --output jsonpath="{.status.url}"| awk -F/ '{print $3}'`

curl -H "Host: ${DOMAIN_NAME}" http://${GATEWAY_IP}
```
测试结果：
从下面的命令输出结果可以看到现在返回的是 v1 的信息，说明请求打到 v1 上面了。
```bash
└─# ./run-test.sh
Welcome to the v1 app!
```

## 灰度 20% 的流量到 v2

创建 v2 revision `revision-v2.yaml` 文件，内容如下：

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: traffic-example
  namespace: default
spec:
  template:
    metadata:
      name: traffic-example-v2
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1
        env:
          - name: RESOURCE
            value: v2
        readinessProbe:
          httpGet:
            path: /
  traffic:
  - tag: v1
    revisionName: traffic-example-v1
    percent: 80
  - tag: v2
    revisionName: traffic-example-v2
    percent: 20
  - tag: latest
    latestRevision: true
    percent: 0
```
我们对比一下 v1 版本和 v2 版本可以发现，v2 版本的 Service 中增加了 `traffic:` 的配置。在 traffic 中指定了每一个 Revision。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1573022880437-a94cea24-eb60-4718-9ab2-139d5d440077.png) 
执行 `kubectl apply -f revision-v2.yaml` 安装 v2 版本的配置。然后多次执行`for ((i=1; i<=10; i++)); do ./run-test.sh; done`这条命令就能看到现在返回的结果中 v1 和 v2 的比例基本是 8:2  的比例。下面这是我真实测试的结果。

```bash
└─# for ((i=1; i<=10; i++)); do ./run-test.sh; done
Welcome to the v1 app!
Welcome to the v2 app!
Welcome to the v1 app!
Welcome to the v1 app!
Welcome to the v1 app!
Welcome to the v1 app!
Welcome to the v2 app!
Welcome to the v1 app!
Welcome to the v2 app!
Welcome to the v2 app!
```

## 提前验证 Revision 
上面展示的 v2 的例子，在创建 v2 的时候直接就把流量分发到 v2 ，如果此时 v2 有问题就会导致有 20% 的流量异常。下面我们就展示一下如何在转发流量之前验证新的 revision 服务是否正常。我们再创建一个 v3 版本。
创建一个 `revision-v3.yaml` 的文件，内容如下：

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: traffic-example
  namespace: default
spec:
  template:
    metadata:
      name: traffic-example-v3
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1
        env:
          - name: RESOURCE
            value: v2
        readinessProbe:
          httpGet:
            path: /
  traffic:
  - tag: v1
    revisionName: traffic-example-v1
    percent: 80
  - tag: v2
    revisionName: traffic-example-v2
    percent: 20
  - tag: latest
    latestRevision: true
    percent: 0
```

执行 `kubectl apply -f revision-v3.yaml` 部署 v3 版本。然后查看一下 Revision 情况：
```bash
└─# kubectl get revision -l "serving.knative.dev/service=traffic-example"
NAME                 CONFIG NAME       K8S SERVICE NAME     GENERATION   READY   REASON
traffic-example-v1   traffic-example   traffic-example-v1   1            True
traffic-example-v2   traffic-example   traffic-example-v2   2            True
traffic-example-v3   traffic-example   traffic-example-v3   3            True
```
可以看到现在已经创建出来了三个 Revision 。
此时我们再看一下 stock-service-example 的真实生效：
```bash
└─# kubectl get ksvc traffic-example -o yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  annotations:
...
status:
...
  
  traffic:
  - latestRevision: false
    percent: 80
    revisionName: traffic-example-v1
    tag: v1
    url: http://v1-traffic-example.default.knative.kuberun.com
  - latestRevision: false
    percent: 20
    revisionName: traffic-example-v2
    tag: v2
    url: http://v2-traffic-example.default.knative.kuberun.com
  - latestRevision: true
    percent: 0
    revisionName: traffic-example-v3
    tag: latest
    url: http://latest-traffic-example.default.knative.kuberun.com
  url: http://traffic-example.default.knative.kuberun.com
```
可以看到 v3 Revision 虽然创建出来了，但是因为没有设置 traffic，所以并不会有流量转发。此时你执行多少次 `./run-test.sh` 都不会得到 v3 的输出。
在 Service 的 status.traffic 配置中可以看到 latest Revision 的配置：
```yaml
  - latestRevision: true
    percent: 0
    revisionName: traffic-example-v3
    tag: latest
    url: http://latest-traffic-example.default.knative.kuberun.com
```
每一个 Revision 都有一个自己的 URL，所以只需要基于 v3 Revision 的 URL 发起请求就能开始测试了。
我已经写好了一个测试脚本，你可以把下面这段脚本保存在 `latest-run-test.sh` 文件中，然后执行这个脚本就能直接发起到 latest 版本的请求：
```bash
#!/bin/bash
export INGRESSGATEWAY=istio-ingressgateway
export GATEWAY_IP=`kubectl get svc $INGRESSGATEWAY --namespace istio-system --output jsonpath="{.status.loadBalancer.ingress[*]['ip']}"`
export DOMAIN_NAME=`kubectl get route ${SVC_NAME} --output jsonpath="{.status.url}"| awk -F/ '{print $3}'`

export LAST_DOMAIN=`kubectl get ksvc traffic-example --output jsonpath="{.status.traffic[?(@.tag=='latest')].url}"| cut -d'/' -f 3`

curl -H "Host: ${LAST_DOMAIN}" http://${GATEWAY_IP}
```
测试 v3 版本如果没问题就可以把流量分发到 v3 版本了。
下面我们再创建一个文件 `revision-v3-2.yaml` , 内容如下：
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: traffic-example
  namespace: default
spec:
  template:
    metadata:
      name: traffic-example-v3
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1
        env:
          - name: RESOURCE
            value: v3
        readinessProbe:
          httpGet:
            path: /
  traffic:
  - tag: v1
    revisionName: traffic-example-v1
    percent: 80
  - tag: v2
    revisionName: traffic-example-v2
    percent: 10
  - tag: v3
    revisionName: traffic-example-v3
    percent: 10
  - tag: latest
    latestRevision: true
    percent: 0
```
用 vimdiff 看一下 revision-v3.yaml 和 revision-v3-2.yaml 的区别：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1573026591696-01f9db15-44ce-4ec6-a91b-e41663e342f3.png) 

revision-v3-2.yaml 增加了到 v3 的流量转发。此时执行 `for ((i=1; i<=10; i++)); do ./run-test.sh; done` 可以看到 v1、v2 和 v3 的比例基本是：8:1:1 
```bash
└─# for ((i=1; i<=10; i++)); do ./run-test.sh; done
Welcome to the v3 app!
Welcome to the v1 app!
Welcome to the v1 app!
Welcome to the v1 app!
Welcome to the v2 app!
Welcome to the v1 app!
Welcome to the v1 app!
Welcome to the v1 app!
Welcome to the v1 app!
Welcome to the v1 app!
```

## 版本回滚
Knative Service 的 Revision 是不能修改的，每次 Service Spec 的更新创建的 Revision 都会保留在 kube-apiserver 中。如果应用发布到某个新版本发现有问题想要回滚到老版本的时候只需要指定相应的 Revision，然后把流量转发过去就行了。

## 小结
Knative Service 的灰度、回滚都是基于流量的。Workload(Pod) 是根据过来的流量自动创建出来的。所以在 Knative Serving 模型中流量是核心驱动。这和传统的应用发布、灰度模型是有区别的。

假设有一个应用 app1 ，传统的做法首先是设置应用的实例个数( Kubernetes 体系中就是 Pod )，我们假设实例个数是 10 个。如果要进行灰度发布，那么传统的做法就是先发布一个 Pod，此时 v1 和 v2 的分布方式是：v1 的 Pod 9个，v2 的 Pod 1 个。如果要继续扩大灰度范围的话那就是 v2 的 Pod 数量变多，v1 的 Pod 数量变少，但总的 Pod 数量维持 10 个不变。

在 Knative Serving 模型中 Pod 数量永远都是根据流量自适应的，不需要提前指定。在灰度的时候只需要指定流量在不同版本之间的灰度比例即可。每一个 Revision 的实例数都是根据流量的大小自适应，不需要提前指定。

从上面的对比中可以发现 Knative Serving 模型是可以精准的控制灰度影响的范围的，保证只灰度一部分流量。而传统的模型中 Pod 灰度的比例并不能真实的代表流量的比例，是一个间接的灰度方法。
