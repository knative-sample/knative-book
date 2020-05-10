---
title: "手动安装 Knative"
date: 2020-5-10T15:26:15Z
draft: false
weight: 30
---

本章主要介绍如何在已有 Kubernetes 集群上执行 Knative 的自定义安装。Knative的模块化组件可以允许您安装所需的组件。
## 准备工作
- 本安装操作中的步骤bash适用于MacOS或Linux环境。对于Windows，某些命令可能需要调整。
- 本安装操作假定您具有现有的Kubernetes集群，可以在其上轻松安装和运行Alpha级软件。
- Knative需要Kubernetes集群v1.14或更高版本，以及可兼容 kubectl。

## 安装Istio
Knative 依赖 Istio 进行流量路由和入口。您可以选择注入 Istio sidecar 并启用 Istio 服务网格，但是并非所有Knative组件都需要它。
如果您的云平台提供了托管的Istio安装，则建议您以这种方式安装Istio，除非您需要自定义安装功能。
如果您希望手动安装Istio，或者云提供商不提供托管的Istio安装，或者您要使用Minkube或类似的本地安装Knative，请参阅[《 安装Istio for Knative》指南](https://github.com/knative/docs/blob/master/docs/install/installing-istio.md)。
***注意：可以使用 [Ambassador](https://www.getambassador.io/)和 [Gloo](https://gloo.solo.io/)替代Istio。 ***

## 安装Knative组件
每个Knative组件必须单独安装。您可以根据需要决定安装哪些组件和内置监控插件。
***注意：如果首次尝试安装失败，请尝试重新运行命令。很可能会在第二次尝试中成功。***

### 选择Knative安装文件
可以使用以下 Knative 安装文件：
- Serving 部分
   - https://github.com/knative/serving/releases/download/{{ <版本>}}/serving.yaml
   - https://github.com/knative/serving/releases/download/{{ <版本>}}/serving-cert-manager.yaml

- 内置监控插件
  - https://github.com/knative/serving/releases/download/{{ <版本>}}/monitoring.yaml
  - https://github.com/knative/serving/releases/download/{{ <版本>}}/monitoring-logs-elasticsearch.yaml
  - https://github.com/knative/serving/releases/download/{{ <版本>}}/monitoring-metrics-prometheus.yaml
  - https://github.com/knative/serving/releases/download/{{ <版本>}}/monitoring-tracing-jaeger.yaml
  - https://github.com/knative/serving/releases/download/{{ <版本>}}/monitoring-tracing-jaeger-in-mem.yaml
  - https://github.com/knative/serving/releases/download/{{ <版本>}}/monitoring-tracing-zipkin.yaml
  - https://github.com/knative/serving/releases/download/{{ <版本>}}/monitoring-tracing-zipkin-in-mem.yaml

- Eventing 组件
  - https://github.com/knative/eventing/releases/download/{{ <版本>}}/release.yaml
  - https://github.com/knative/eventing/releases/download/{{ <版本>}}/eventing.yaml
  - https://github.com/knative/eventing/releases/download/{{ <版本>}}/in-memory-channel.yaml

- Eventing 事件源
  - https://github.com/knative/eventing-contrib/releases/download/{{ <版本>}}/github.yaml
  - https://github.com/knative/eventing-contrib/releases/download/{{ <版本>}}}/camel.yaml
  - https://github.com/knative/eventing-contrib/releases/download/{{ <版本>}}}/gcppubsub.yaml
  - https://github.com/knative/eventing-contrib/releases/download/{{ <版本>}}}/kafka.yaml
  - https://github.com/knative/eventing-contrib/releases/download/{{ <版本>}}/kafka-channel.yaml
   
### 安装Knative
1. 如果要从 Knative 0.3.x 升级，请执行以下操作：将 domain 和静态 IP 地址更新为与 istio-ingressgateway 关联， 而不是 knative-ingressgateway。然后运行以下命令清理剩余的资源：
```
kubectl delete svc knative-ingressgateway -n istio-system
kubectl delete deploy knative-ingressgateway -n istio-system
```
如果安装了Knative Eventing Sources组件，则还需要在升级之前删除以下资源：

```
kubectl delete statefulset/controller-manager -n knative-sources
```

2. 要安装 Knative 组件或插件，请在kubectl apply命令中指定文件名 。为了防止由于资源安装顺序导致安装失败，请首先运行带有该 -l knative.dev/crd-install=true 标志的安装命令，然后再次运行没有 --selector 标志的安装命令。
**示例安装命令**:
- 如果需要使用内置监控插件，安装 Knative Serving 组件，请运行以下命令：
   - 仅安装CRD：
```
kubectl apply --selector knative.dev/crd-install=true \
  --filename https://github.com/knative/serving/releases/download/{{ <版本> }}/serving.yaml \
  --filename https://github.com/knative/serving/releases/download/{{ <版本> }}/monitoring.yaml
```
   - 删除--selector knative.dev/crd-install=true 标志，然后运行命令以安装Serving组件和监控插件：
```
kubectl apply --filename https://github.com/knative/serving/releases/download/{{ <版本> }}/serving.yaml \
  --filename https://github.com/knative/serving/releases/download/{{ <版本> }/monitoring.yaml
```
- 如果不使用内置监控插件的情况下安装所有Knative组件，请运行以下命令。
   - 仅安装CRD：
```
kubectl apply --selector knative.dev/crd-install=true \
  --filename https://github.com/knative/serving/releases/download/{{ <version> }}/serving.yaml \
  --filename https://github.com/knative/eventing/releases/download/{{ <version> }}/release.yaml
```
   - 删除--selector knative.dev/crd-install=true 标志，然后运行命令以安装所有Knative组件，包括Eventing资源：
   ```
kubectl apply --filename https://github.com/knative/serving/releases/download/{{ <版本> }}/serving.yaml \
  --filename https://github.com/knative/eventing/releases/download/{{ <版本> }}/release.yaml
```
3. 根据选择安装的内容，通过运行以下一个或多个命令来查看安装状态：
```
kubectl get pods --namespace knative-serving
kubectl get pods --namespace knative-eventing
```
4. 如果安装了内置监控插件，请运行以下命令：
```
kubectl get pods --namespace knative-monitoring
```
## 其它
考虑到国内用户有可能拉取不断外部镜像，安装文件可参考：https://github.com/knative-sample/knative-release/tree/v0.10.0

