---
title: "在阿里云上一键安装 Knative"
date: 2020-5-10T15:26:15Z
draft: false
weight: 20
---

本文介绍一下如何在阿里云容器服务中一键安装 Knative。
## 前提条件
- 您已经成功创建一个 Kubernetes 集群，参见[创建Kubernetes集群](https://help.aliyun.com/document_detail/86488.html?spm=a2c4g.11186623.2.10.571b21d6Lo30j5#CS-user-guide-kubernetes)
- 部署成功 Istio, 参见[部署Istio](https://help.aliyun.com/document_detail/89805.html?spm=a2c4g.11186623.6.709.3185450dtIjfEm)
- 仅支持 Kubernetes 版本 1.14 及以上的集群。 只支持**标准托管**以及**标准专有** Kubernetes 集群
- 一个集群中，worker节点数量需要大于等于3个，保证资源充足可用
## 操作步骤
1. 登录 [容器服务管理控制台](https://cs.console.aliyun.com)
2. 单击左侧导航栏中的**Knative**>**组件管理**，进入**组件管理**页面
3. 点击**一键部署**按钮
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572869762654-82a2c12f-014f-4bec-aefb-6014222bdce7.png) 
4. 选择需要安装的 Knative 组件（Tekton 组件、Serving 组件和 Eventing 组件），点击**部署**
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572869787272-2cdff85b-fd05-4cd2-8ee4-58c8d2b1ae25.png) 
## 执行结果
部署完成之后，可以看到部署结果信息。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572869863110-6e5e4cb4-d637-41a1-bd1d-e5f3adc5e814.png) 
点击**进入组件管理**，查看组件信息。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1572870278281-2144c435-126e-45a2-8c37-7e4e4d576f4d.png) 

## 总结
在阿里云容器服务中提供了自定义安装 Knative 组件以及 addon 组件，让你轻松安装 Knative。