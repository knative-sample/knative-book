---
title: "使用 GitHub 事件源"
date: 2020-5-10T15:26:15Z
draft: false
weight: 30
description: ""
---

## 前提条件
- 您已经成功创建一个 Kubernetes 集群
- 部署成功 Knative 
- 部署成功 Serving 组件， 并且完成域名配置。
- 部署成功 Eventing 组件

## 操作步骤 
### 部署 GitHub 事件源。
1. 登录 [容器服务管理控制台](https://cs.console.aliyun.com)。
2. 在 Kubernetes 菜单下，单击左侧导航栏的**Knative** > **组件管理**， 选择安装 GitHub addon 组件，如图所示
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573028455914-6954cd0e-6610-4a99-b4f3-6ff2ec49fa87.png) 
### 创建服务
1. 单击左侧导航栏的**Knative** > **服务管理**，进入**服务管理**页面。
2. 右上角单击**创建服务**。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1563760587315-3608203b-7efe-4bf2-a033-8dfd5c5d5c0f.png) 
4. 设置部署集群、命名空间、服务名称。选择所要使用的镜像和镜像的版本，如图：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1566823432696-9f63d5cd-9091-4a9a-b72e-cfb2e4cd3430.png) 
- 镜像名称：registry.cn-shanghai.aliyuncs.com/knative-release/eventing-sources-event_display
- 镜像版本：bf45b3eb1e7fc4cb63d6a5a6416cf696295484a7662e0cf9ccdf5c080542c21d

5. 点击 **创建**按钮

### 创建 GitHub Token
创建 [Personal access tokens](https://github.com/settings/tokens), 用于访问 GitHub API。token 的名称可以任意设置。`Source` 需要开启 `repo:public_repo` 和 `admin:repo_hook` , 以便通过公共仓库触发 Event 事件，并为这些公共仓库创建 webhooks 。
下面是设置一个 "GitHubSource Sample" token 的示例。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1559396080243-c86eccce-8094-44d1-83a8-bf05dda974f1.png) 

secretToken内容可以通过下述方式生成随机字符串：
```
head -c 8 /dev/urandom | base64
```

更新 `githubsecret.yaml` 内容。如果生成的是 `personal_access_token_value` token, 则需要设置 `secretToken` 如下：

```
apiVersion: v1
kind: Secret
metadata:
  name: githubsecret
type: Opaque
stringData:
  accessToken: personal_access_token_value
  secretToken: asdfasfdsaf
```
执行命令使其生效：

```
kubectl --namespace default apply --filename githubsecret.yaml
```
### 创建 GitHub 事件源
为了接收 GitHub 产生的事件， 需要创建 GitHubSource 用于接收事件。

```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: GitHubSource
metadata:
  name: githubsourcesample
spec:
  eventTypes:
    - pull_request
  ownerAndRepository: <YOUR USER>/<YOUR REPO>
  accessToken:
    secretKeyRef:
      name: githubsecret
      key: accessToken
  secretToken:
    secretKeyRef:
      name: githubsecret
      key: secretToken
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: github-event-display
```
执行 kubectl 命令：

```
kubectl --namespace default apply --filename github-source.yaml
```

### 验证结果
在 GitHub repository 中 Settings->Webhooks 查看会有一个验证成功的 Hook URL。注意：域名需要备案，否则无法进行访问。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1559455927328-28c67206-d122-45f0-beea-7f5417c6e940.png) 

在 GitHub repository 中 创建一个 `pull request `，会产生Event事件，然后在 Knative Eventing 可以查看：

```
kubectl --namespace default get pods
kubectl --namespace default logs github-event-display-XXXX user-container
```
我们可以看到类似下面的事件结果：

```
2018/11/08 18:25:34 Message Dumper received a message: POST / HTTP/1.1
Host: github-event-display.knative-demo.svc.cluster.local
Accept-Encoding: gzip
Ce-Cloudeventsversion: 0.1
Ce-Eventid: a8d4cf20-e383-11e8-8069-46e3c8ad2b4d
Ce-Eventtime: 2018-11-08T18:25:32.819548012Z
Ce-Eventtype: dev.knative.source.github.pull_request
Ce-Source: https://github.com/someuser/somerepo/pull/1
Content-Length: 21060
Content-Type: application/json
User-Agent: Go-http-client/1.1
X-B3-Parentspanid: b2e514c3dbe94c03
X-B3-Sampled: 1
X-B3-Spanid: c85e346d89c8be4e
X-B3-Traceid: abf6292d458fb8e7
X-Envoy-Expected-Rq-Timeout-Ms: 60000
X-Envoy-Internal: true
X-Forwarded-For: 127.0.0.1, 127.0.0.1
X-Forwarded-Proto: http
X-Request-Id: 8a2201af-5075-9447-b593-ec3a243aff52

{"action":"opened","number":1,"pull_request": ...}
```
## 总结
通过 Knative Eventing 对接 GitHub 事件源，可以感知代码 Merge、Issue 提交等事件，进而针对这些事件触发对于的服务进行处理。
