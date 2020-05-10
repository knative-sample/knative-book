---
title: "调用链管理"
date: 2020-5-10T15:26:15Z
draft: false
weight: 20
description: ""
---

阿里云链路追踪 Tracing Analysis 为分布式应用的开发者提供了完整的调用链路还原、调用请求量统计、链路拓扑、应用依赖分析等工具。本文介绍了如何在 Knative 上实现 Tracing 分布式追踪,  以帮助开发者快速分析和诊断 Knative 中部署的应用服务。
## 前提条件
- 您已经成功创建一个 Kubernetes 集群，参见[创建Kubernetes集群](https://help.aliyun.com/document_detail/86488.html?spm=a2c4g.11186623.2.10.571b21d6Lo30j5#CS-user-guide-kubernetes)
- 部署 Istio，参见[部署Istio](https://help.aliyun.com/document_detail/89805.html?spm=a2c4g.11186623.6.709.3185450dtIjfEm)，这里需要注意以下几点：
  - Pilot 设置跟踪采样百分比，推荐设置大一些（如：100），便于数据采样
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560320339999-5fc5d4d3-bc42-4859-bcc4-9a9912712b67.png) 
  - 启用[阿里云链路追踪服务](https://tracing-analysis.console.aliyun.com/?spm=5176.2020520152.0.0.2fc916ddrsNrsH)，查看 token 点击 on, 客户端采集工具选择 **zipkin** （目前仅支持使用 zipkin 的 /api/v1/spans 接口访问）， 选择集群所在 Region 的接入点（推荐使用内网接入地址），以华南1 Region 为例如下: 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573096041282-9a2643d6-e73c-428c-8b02-6039d554671f.png) 

  - 选择**启用链路追踪**。设置链路追踪服务地址
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560331555972-3e3359e5-261c-4aa2-998a-bf55db75c1dc.png) 
- 部署成功 Knative , 参见[部署Knative](https://help.aliyun.com/document_detail/121509.html)
- 部署成功 Serving 组件

## 调用链追踪 
1. 选择命名空间设置如下标签启用 Sidecar 自动注入：istio-injection=enabled。通过这种方式就注入了 Istio 的 envoy 代理（proxy）容器， Istio 的 envoy 代理拦截流量后会主动上报 trace 系统。以设置 default 命名空间为例：

```
kubectl label namespace default istio-injection=enabled
```
2. 部署 Knative Service 服务。参考 Serving Hello World 章节
3. 访问 Hello World 示例服务。

```
$ curl -H "Host: helloworld-go.default.example.com" http://112.124.XX.XX
Hello Go Sample v1!
```
4. 登录[阿里云链路追踪服务控制台](https://tracing-analysis.console.aliyun.com/)， 选择**应用列表**，可以查看对应应用的 tracing 信息。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560326238195-38dd2fff-3713-4233-a241-7406d0a00f5b.png) 
5. 选择应用，点击查看**应用详情**，可以看到服务调用的平均响应时间。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560326140105-47136a02-7c52-4a49-ae31-71e40a980700.png) 

## 总结
在 Knative 中，推荐开启阿里云链路追踪服务。通过 Tracing 不仅有助于复杂应用服务之间的问题排查及效率优化，同时也是 Knative 的最佳实践方式。
