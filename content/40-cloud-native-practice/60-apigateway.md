---
title: "基于 APIGateway 打造生产级别的 Knative 服务"
date: 2020-5-10T15:26:15Z
draft: false
weight: 60
description: ""
---

在实际应用中，通过 APIGateway（即 API 网关），可以为内部服务提供保护，提供统一的鉴权管理，限流、监控等能力，开发人员只需要关注内部服务的业务逻辑即可。本文就会介绍一下如何通过阿里云 API 网关结合内网 SLB，将 Knative 服务对外发布，以打造生产级别的 Knative 服务。

## 关于阿里云 API 网关
阿里云 API 网关为您提供完整的 API 托管服务，辅助用户将能力、服务、数据以 API 的形式开放给合作伙伴，也可以发布到 API 市场供更多的开发者采购使用。
- 提供防攻击、防重放、请求加密、身份认证、权限管理、流量控制等多重手段保证 API 安全，降低 API 开放风险
- 提供 API 定义、测试、发布、下线等全生命周期管理，并生成 SDK、API 说明文档，提升 API 管理、迭代的效率
- 提供便捷的监控、报警、分析、API 市场等运维、运营工具，降低 API 运营、维护成本

## 基于阿里云 API 网关发布服务
### 绑定 Istio 网关到内网SLB
创建内网SLB，绑定 Istio 网关应用。可以直接通过下面的 yaml 创建内网 SLB：
```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/alicloud-loadbalancer-address-type: "intranet"
  labels:
    app: istio-ingressgateway
    istio: ingressgateway
  name: istio-ingressgateway-intranet
  namespace: istio-system
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: status-port
    port: 15020
    protocol: TCP
    targetPort: 15020
  - name: http2
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
  - name: tls
    port: 15443
    protocol: TCP
    targetPort: 15443
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  sessionAffinity: None
  type: LoadBalancer
```
创建完成之后，可以在登录[阿里云容器服务控制台](https://cs.console.aliyun.com)，进入 【路由和工作负载】菜单，选择 `istio-system` 命名空间，可以查看到所创建的内网 SLB 信息：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567652311381-3813fb0b-818f-4a70-a4a1-7aeaffbf5624.png) 

此处内网 SLB 地址为：192.168.0.23

### 创建 Knative 服务
登录阿里云容器服务控制台，[创建 Knative 服务](https://help.aliyun.com/document_detail/126413.html)。
这里我们创建 helloworld 服务，如图所示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567648371907-5018b365-924f-413f-9050-5a282b1226e7.png) 
验证一下服务是否可以访问：
```
[root@iZbp1c1wa320d487jdm78aZ ~]# curl -H "Host:helloworld.default.example.com" http://192.168.0.23
Hello World!
```

### 配置 API 网关
接下来进入重头戏，如何配置 API 网关与 Knative Service 进行访问。
#### 创建分组
由于 API 需要归属分组，我们首先创建分组。登录[阿里云API 网关控制台](https://apigateway.console.aliyun.com)，开放API->分组管理：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567653666476-3d4434ef-4bb9-4c4b-a217-d02b00fab58e.png) 
点击【创建分组】，选择共享实例（VPC）
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567652879838-2ef79fc4-bb4b-4878-a244-3edfa788596a.png) 

创建完成之后，我们需要在分组详情中开启公网域名，以进行公网服务访问：可以通过 `1` 开启公网二级域名进行测试，或者通过 `2` 设置独立域名。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567653028874-93481f3b-c72b-42c9-a6e6-79d4ed2a3623.png) 
这里我们开启公网二级域名进行测试访问，开启后如图所示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567653181947-770573b0-9868-41a2-bdbd-7886de98b057.png) 
#### 创建 VPC 授权
由于我们是访问K8s VPC内的服务，需要创建 VPC 授权。选择 开放API->VPC 授权：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567655135954-799bf4c9-d4c5-45db-a1da-76992d2bf165.png) 
点击【创建授权】，设置VPC Id以及内网SLB实例Id。这里创建 `knative-test` VPC 授权
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567655365765-2fedd478-a78f-4b74-b7e2-78d3580fd9f8.png) 
#### 创建应用
创建应用用于`阿里云APP` 身份认证。该认证要求请求者调用该 API 时，需通过对 APP 的身份认证。这里我们创建 `knative` 应用。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567654314489-965356b0-9b2a-4388-8b33-f03f12d56a70.png) 
#### 创建 API
登录阿里云API 网关控制台，开放API->API列表，选择【创建API】。关于创建API，详细可参考：[创建API](https://help.aliyun.com/document_detail)。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567653897838-dae2bfef-8a2b-462c-b593-525b8a56e202.png) 
接下来我们输入【基本信息】，选择安全认证：阿里云APP，AppCode认证可以选择：允许AppCode认证（Header & Query）。具体AppCode认证方式可以参考：[使用简单认证（AppCode）方式调用API](https://help.aliyun.com/document_detail/115437.html)
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567654781103-3564e9af-784d-49ac-9e3c-e5174f162eab.png) 
点击下一步，定义API请求。协议可以选择HTTP和HTTPS， 请求Path可设置`/`。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567654742646-88bca315-5fd7-482e-a123-c53bb864afcd.png) 
点击下一步，定义API后端服务。后端服务类型我们设置为VPC，设置VPC授权名称等。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567745881336-b4b22099-4338-422b-88d8-bd1346765ca1.png) 
设置`常量参数`，其中后端参数名称：Host，参数值：helloworld.default.example.com，参数位置：Header。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567655633316-04f0b501-be45-4bd7-95f7-6a62f70ebc18.png) 
点击下一步，完成创建。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567655778284-076f746a-8ace-423f-aa74-044c3b2252c1.png) 
#### 发布 API
创建完成之后，可直接进行发布。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567655799602-5144650b-30f0-404c-8cba-63ba4a85b4a4.png)
选择 `线上`，点击【发布】
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567655844383-8a4add12-a1d5-4caf-b7dc-19162f8eb0bf.png) 
#### 验证 API
发布完成之后，我们可以在【API列表】中看到当前API：线上 （运行中）
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567656023971-fcc6cfa8-90cc-49ba-81c2-f417a7263d5c.png) 
在调用API测试之前，我们需要对该API进行应用授权，进入API详情，选择【授权信息】
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567659986941-ab4cda42-3c10-49d7-8893-babf4d1e7538.png) 
点击【添加授权】，这里我们选择上面创建的`knative`应用进行授权
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567660043015-d26d1a9e-2a86-409e-9ee6-0efe7a8129b0.png) 
接下来我们进行验证API，点击在API详情中，选择【调试API】，点击【发送请求】，可以看到测试结果信息：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1567660384905-929f92a8-8f40-4f6b-bb55-565d6b95f725.png) 

至此，我们通过阿里云 API 网关将 Knative 服务发布完成。

## 小结
通过上面的介绍，相信大家对如何通过阿里云 API 网关将 Knative 服务对外发布有了初步的了解。在实际生产中我们对Serverless 服务的访问安全、流控、监控运维等要求是不必可少的，而通过阿里云 API 网关恰好可以对 Knative 服务提供保驾护航能力。通过阿里云 API 网关可以对 API 服务配置：
- 流量控制
- 访问鉴权
- 日志监控
- API 全生命周期管理 : 测试、发布、下线

正是通过这些能力，阿里云 API 网关为 Knative 提供生产级别的服务。欢迎有兴趣的同学一起交流。

