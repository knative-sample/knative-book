---
title: "基于 MNS 与 OSS 实现人脸图片识别"
date: 2020-5-10T15:26:15Z
draft: false
weight: 50
description: ""
---

标准 Serverless 框架和人脸识别服务结合会产生怎样的火花？本文介绍如何通过 Knative 实现人脸识别服务，看看能否给你带来不一样的体验。
## 场景
通过 OSS 控制台上传照片，MnsOss 事件源接收图片上传的事件信息，发送到 Knatvie Eventing，通过Broker/Trigger事件处理模型之后，接着触发 Knative Serving 中的人脸识别服务进行分析。最后把分析之后的图片回传到 OSS。
![image](https://yqfile.alicdn.com/5c89f6dacb16aa5d75d49183cc0d759b1fe717b3.png)


## 准备
- 安装 Knative Serving 和 Eventing
- 安装 Knative MnsOssSource 事件源服务
容器服务控制台->Knative->组件管理，选择 MnsOss 安装
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573026324190-73e06a6a-4c00-4869-92e7-4f990d8b7ecd.png) 

- 在 OSS 控制台创建 Bucket，参见[创建存储空间](https://help.aliyun.com/document_detail/31896.html)
这里注意创建 Bucket 时需要设置为**公共读**，以提供人脸识别服务进行访问图片。
- 已开通MNS服务，参见[开通MNS服务](https://help.aliyun.com/document_detail/27423.html?spm=a2c4g.11186623.6.554.46057f87UWfzGx)
- 已开通人脸识别服务，参见[开通人脸识别服务](https://help.aliyun.com/document_detail/56811.html?spm=a2c4g.11174283.6.545.75735d0dERco5h)

## 操作
### 创建 OSS 事件通知
选择 Bucket, 点击**事件通知**页签
![image](https://yqfile.alicdn.com/cf870f7448e5aa72ab43cd20ed5e13c368b6fe37.png)

创建规则：
![image](https://yqfile.alicdn.com/4d1d826da02fb71e5784ca69c86a32427d2bb60d.png)


创建完成之后，会在MNS中生成相应的Topic：
![image](https://yqfile.alicdn.com/d67f1aaae26be5e3a46915fb8fcbe9bdecb205f9.png)


获取公网Topic访问连接：
![image](https://yqfile.alicdn.com/b2faf9c20412250ab55f70aa30a8bd166c8c2b0d.png)

这里我们选择公网访问连接：https://xxxx.mns.cn-shanghai.aliyuncs.com/

### 创建 Mns Token
获取上面的公网 Topic 访问连接以及ak, sk信息。按照下面的格式进行base64进行编码处理，生成访问Token。
```
# echo '{ "url":"https://xxxx.mns.cn-shanghai.aliyuncs.com/", "accessKeyId":"xxx","accessKeySecret":"xx" }' | base64
```
设置 `mnsoss-secret.yaml` 内容。则需要设置 `mns` 如下：

```
apiVersion: v1
kind: Secret
metadata:
  name: mnsoss-secret
type: Opaque
data:
  mns: eyAidXJsIjoiaHR0cHM6Ly94eHh4Lm1ucy5jbi1zaGFuZ2hhaS5hbGl5dW5jcy5jb20vIiwgImFjY2Vzc0tleUlkIjoieHh4IiwiYWNjZXNzS2V5U2VjcmV0IjoieHgiIH0K
```
执行命令使其生效：

```
kubectl apply -f mnsoss-secret.yaml
```
### 创建 Service Account及角色绑定
设置 `mnsoss-sa.yaml` 内容。
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eventing-sources-mnsoss
subjects:
- kind: ServiceAccount
  name: mnsoss-sa
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: eventing-sources-mnsoss-controller

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mnsoss-sa
```
执行命令使其生效：

```
kubectl apply -f mnsoss-sa.yaml
```

### 设置istio egress（若当前命名空间下启用了istio注入）

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: alimns-ext
spec:
  hosts:
  - "*.aliyuncs.com"
  ports:
  - number: 443
    name: https
    protocol: HTTPS
```


### 创建 Broker

```
kubectl label namespace default knative-eventing-injection=enabled
```
### 创建 MnsOss 事件源
为了接收 MnsOss 产生的事件， 需要创建 MnsOssSource 用于接收事件。mnsoss-source.yaml如下：

```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: MnsOssSource
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: mnsoss-face
spec:
  # Add fields here
  serviceAccountName: mnsoss-sa
  accessToken:
    secretKeyRef:
      name: mnsoss-secret
      key: mns
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Broker
    name: default
  topic: mns-en-topics-oss-face-image-2381221888dds9129
```
参数说明：

```
topic：表示 MNS 主题名称
```

执行 kubectl 命令：

```
kubectl  apply -f mnsoss-source.yaml
```
### 创建 Knative Service
为了验证 `MnsOssSource` 是否可以正常工作，可以这里使用人脸识别的的 Knative Service 示例。service.yaml如下：

```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: face-recognition
  namespace: default
spec:
  template:
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/face-recognition:v0.2.7
        command:
        - '/app/face-recognition'
        - --configpath=/app/config
        env:
        - name: ACCESSKEY_ID
          value: "xxx"
        - name: ACCESSKEY_SECRET
          value: "xxx"
        - name: UPLOAD_OSS_PATH
          value: "face-image/target"
```
env参数说明：

```
UPLOAD_OSS_PATH：表示目标图片的存放位置
ACCESSKEY_ID：用户ak信息
ACCESSKEY_SECRET：用户sk信息
```

执行以下命令创建 Service。

```
kubectl apply -f service.yaml
```
### 创建 Trigger
创建 Trigger， 订阅事件信息。trigger.yaml如下：
```
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: face-trigger
  namespace: default
spec:
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: face-recognition
```
执行 kubectl 命令：

```
kubectl apply -f trigger.yaml
```

### 验证
通过 OSS 上传人脸图片。会在目标图片的存放位置生成人脸识别结果图片。
识别前：
![46985432075_56c3190d32_o](https://yqfile.alicdn.com/13e7750f7bf68d837b24076f79e6b7455527384e.jpeg)

识别结果：
![46985432075_56c3190d32_o](https://yqfile.alicdn.com/88cd6298d338e601326c051def111216f61e157e.jpeg)


## 总结
从事件源到 Eventing，再到 Serving 进行服务处理，通过 Knative 实现方式是不是给你带来了不一样的体验。你可以结合实际的场景使用 Knative 打造属于自己的识别系统。

## 参考文档
- mnsoss代码示例：https://github.com/knative-sample/mnsoss/tree/b1.0
- 代码示例：https://github.com/knative-sample/face-recognition/tree/b1.0
