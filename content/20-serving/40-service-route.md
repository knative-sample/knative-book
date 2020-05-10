---
title: "服务路由管理"
date: 2020-5-10T15:26:15Z
draft: false
weight: 40
description: "calling built-in Shortcodes into your content files."
---


Knative 默认会为每一个 Service 生成一个域名，并且 Istio Gateway 要根据域名判断当前的请求应该转发给哪个 Knative Service。Knative 默认使用的主域名是 example.com，这个域名是不能作为线上服务的。本文我首先介绍一下如何修改 默认主域名，然后再深入一层介绍如何添加自定义域名以及如何根据 path 关联到不同的 Knative Service。

## Knative Serving 的默认域名 example.com 
首先需要部署一个 Knative Service。如果你已经有了一个 Knative 集群，那么直接把下面的内容保存到 login-service.yaml 文件中。然后执行一下 `kubectl apply -f  login-service.yaml  `  即可把 login-service 服务部署到 default namespace 中。
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: login-service
spec:
  template:
    metadata:
      labels:
        app: login-service
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:73fbdd56
          env:
            - name: TARGET
              value: "Login Service"
```

现在我们来看一下 Knative Service 自动生成的域名配置:
```
└─# kubectl -n default get ksvc
NAME    URL                                   LATESTCREATED   LATESTREADY   READY   REASON
login-service   http://login-service.default.example.com   login-service-wsnvc     login-service-wsnvc   True
```

现在使用 curl 指定 Host 就能访问服务了。
- 首先获取到 Istio Gateway IP 
```
└─# kubectl get svc istio-ingressgateway --namespace istio-system --output jsonpath="{.status.loadBalancer.ingress[*]['ip']}"
47.95.191.136
```
- 访问 login-service 服务
```
└─# curl -H "Host: login-service.default.example.com" http://47.95.191.136/
Hello Login Service
```
如果想要在浏览器中访问 login-service 服务需要先做 host 绑定，把域名 login-service.default.example.com 指向 47.95.191.136 才行。这种方式还不能对外提供服务。下面接着介绍一下如何把默认的 example.com 改成我们自己的域名。
## 使用自定义主域名
在阿里云 Knative 用户可以通过控制台配置自定义域名，并基于Path和Header进行路由转发设置。如图所示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572261356163-327e6512-566c-4d8d-97bf-0062817cacc0.png) 
假设我们自己的域名是：knative.kuberun.com，现在执行 `kubectl edit cm config-domain --namespace knative-serving` 如下图所示，添加 knative.kuberun.com 到 ConfigMap 中，然后保存退出就完成了自定义主域名的配置。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573010931575-073b2eef-199d-4e71-8b40-a89f2334652e.png) 
再来看一下 Knative Service 的域名, 如下所示已经生效了。
```
└─# kubectl -n default get ksvc
NAME    URL                                              LATESTCREATED   LATESTREADY   READY   REASON
login-service   http://login-service.default.knative.kuberun.com   login-service-wsnvc     login-service-wsnvc   True
```

 **泛域名解析**
 Knative Service 默认生成域名的规则是 servicename.namespace.use-domain 。所以不同的 namespace 会生成不同的子域名，每一个 Knative Service 也会生成一个唯一的子域名。为了保证所有的 Service 服务都能在公网上面访问到，需要做一个泛域名解析。把 `*.knative.kuberun.com`  解析到 Istio Gateway 47.95.191.136 上面去。如果你是在阿里云(万网)上面购买的域名，你可以通过如下方式配置域名解析：
 ![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1565859153193-f026bb89-ccb5-47e9-940e-86880b15f800.png) 
 
 现在直接通过浏览器访问 [http://login-service.default.knative.kuberun.com/](http://login-service.default.knative.kuberun.com/) 就可以直接看到 login-service 服务了：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573011800077-779099e3-7bec-4256-b01b-dbb782b7bbdc.png) 

### 自定义域名
登录阿里云容器服务控制台，进入【Knative】-【组件管理】，点击 Serving 组件【详情】。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573019330143-28efc83c-7c78-40de-ba15-ccffd7045065.png) 
进入详情之后，选择域名配置，添加自定义域名：test.knative.kuberun.com。点击 【确定】进行保存。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572262317426-486bd2ef-fab9-44fe-955c-0a5f111c6d55.png) 

### 配置路由转发
进入【Knative】-【服务管理】控制台，选择对应的服务。这里我们对 Login-Service 服务 以及 Search-Service 服务分别设置不同的 Path 进行访问。
#### Login-Service 服务路由转发配置
选择  Login-Service 服务， 选择 路由转发 页签，点击 配置， 选择`test.knative.kuberun.com`域名，配置路径：/login。点击 确定 进行保存。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572262396948-f562102a-5892-4632-b238-40a92329b0ed.png) 

接下了继续配置Search-Service 服务路由规则。
#### Search-Service 服务路由转发配置
选择  Search-Service 服务， 选择 路由转发 页签，点击 配置， 选择`test.knative.kuberun.com`域名，配置路径：/search。点击 确定 进行保存。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572262431634-c0a691e5-1246-443a-88e2-19332573a431.png) 
### 服务访问
以上路由转发配置完成之后，我们开始测试一下服务访问：
在浏览器中输入：http://test.knative.kuberun.com/login 可以看到输出：Hello Login Service!
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572262493016-07c15835-be14-4743-8e74-94d01449584b.png) 

在浏览器中输入：http://test.knative.kuberun.com/search 可以看到输出：Hello Search Service!
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572262513018-241672b4-a2b0-4da1-87ff-d5277eb57166.png) 

### 基于 Path + Header 进行路由转发
选择  Login-Service 服务， 选择 `路由转发` 页签，点击 配置，这里我们加上Header 配置：foo=bar。点击 确定 进行保存。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572488430015-0303c6f7-bedb-4cb9-b972-79610d97e779.png) 

访问 http://test.knative.kuberun.com/login 发现服务 404 不可访问。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572262660880-d29f4fdd-3b75-401d-a9d6-f300f166fe2e.png) 

说明基于Header是生效的，下面我们在访问请求中通过 ModHeader 插件配置上Header：foo=bar.
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572488486433-b525e8b7-ff20-4152-98bc-38e64fcc46bd.png) 
配置完成之后，我们再一次访问服务：http://test.knative.kuberun.com/login。 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572262772644-418b2cba-237e-4511-974f-dcf587ccef32.png) 
服务访问 OK。这样我们就完成了基于 Path + Header 路由转发配置

## 总结
以上主要围绕 Knative Service 域名配置展开介绍了 Knative Serving 的路由管理，并且通过阿里云 Knative 控制台让你更轻松、快捷的实现自定义域名及路由规则，以打造生产可用的服务访问。