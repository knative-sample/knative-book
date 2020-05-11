---
title: "综合应用-从代码到生产级别服务"
date: 2020-5-10T15:26:15Z
draft: false
weight: 90
description: ""
---

在实际使用中，我们往往会遇到一些复杂的场景，这时候就需要各个组件之间进行协作处理。例如我们提交源代码之后是否直接可以部署服务到 K8s 中，对这些服务如何进行日志监控及告警设置？ 这写场景对于用户来说是很有吸引力。那么现在就让我们来看一下在 Knative 中如何实现从代码到生产级别服务。
## 场景介绍
现在的场景是这样的：代码构建->事件驱动->服务部署。那么对应到Knative中，需要 Eventing、Tekton 和 Serving 一起协作来实现这个场景。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1565013576022-a07822ab-9fb0-4f06-9d8a-eb23bd4d15a8.png) 
## 准备
- 部署 Knative。参考在阿里云容器服务上[部署Knative](https://help.aliyun.com/document_detail/121509.html)。 
- 部署 Tekton。通过阿里云容器服务控制台，【Knative】->【组件管理】选择 `Tekton` 组件进行安装部署：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573028413465-2421b152-c41d-4fc8-97c7-d452be6e54cc.png) 
- 部署 GitHub 事件源。阿里云容器服务控制台Knative  组件管理中选择安装 GitHub addon 组件，如图所示
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573028455914-6954cd0e-6610-4a99-b4f3-6ff2ec49fa87.png) 

## 从源代码到服务
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1565014966816-7e476d4c-add1-46be-a319-552fed1d5019.png) 
- 修改分支代码，提交 merge request 合并到 master 分支
- Eventing 监听到 merge 事件，发送给 GitHub Trigger 服务
- GitHub Trigger 服务接收事件, 通过Tekton执行代码构建和并通过deployer执行服务部署。GitHub  Trigger 的作用就是解析 GitHub 事件的详细信息，然后转换成 Tekton 资源并且提交到 Kubernetes 中执行 Pipeline。 项目地址：https://github.com/knative-sample/tekton-serving/tree/b1.0。 这个项目中有两个部分： Trigger 和 Deployer，Trigger 的作用是解析 github 事件， 并提交 PipelineRun 定义。Deployer 的作用就是更新 Service 的镜像信息。github source pull_request body 的关键内容如下：
	```json
	{
	  "action": "closed",
		... ...
		"merge_commit_sha": "f37cb28b1777a28cd34ea1f8df1b7ebcc6c16397",
		... ...
		"base": {
		  "ref": "master",
		  ... ...
		  },
		... ...
	}
	```
   - action 表示当前的 pull request 事件细节。创建 pull request 时 action  是 opened ，关闭 pull request 时 action 就是 closed 
   - merge_commit_sha 可以获得 merge commit 的 id
   - base.ref 可以获得 merge request 发生在哪个分支上

本文涉及到的代码与资源文件地址：
- GitHubTrigger 和 Deployer：https://github.com/knative-sample/tekton-serving/tree/b1.0
- 部署示例文件：https://github.com/knative-sample/eventing-tekton-serving/tree/b1.0

接下来我们开始一步步搞起。
### 部署 Tekton 服务
我们看一下创建代码构建 Task 和 部署服务Task
代码构建Task：
```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: source-to-image
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToDockerFile
        description: The path to the dockerfile to build (relative to the context)
        default: Dockerfile
      - name: imageUrl
        description: Url of image repository
      - name: imageTag
        description: Tag to apply to the built image
        default: "latest"
  steps:
    - name: build-and-push
      image: registry.cn-hangzhou.aliyuncs.com/knative-sample/kaniko-project-executor:v0.10.0
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(inputs.params.imageUrl):$(inputs.params.imageTag)
        - --context=/workspace/git-source/$(inputs.params.pathToContext)
      env:
      - name: DOCKER_CONFIG
        value: /builder/home/.docker

```

这里通过 deployer-deployer 执行服务部署，部署服务Task：
```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: image-to-deploy
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToYamlFile
        description: The path to the yaml file to deploy within the git source
      - name: imageUrl
        description: Url of image repository
      - name: imageTag
        description: Tag of the images to be used.
        default: "latest"
  steps:
    - name: deploy
      image: "registry.cn-hangzhou.aliyuncs.com/knative-sample/deployer-deployer:v1_74647e3a-20190807193731"
      args:
        - "--namespace=default"
        - "--serivce-name=hello-sample"
        - "--image=${inputs.params.imageUrl}:${inputs.params.imageTag}"

```
另外需要设置一下镜像仓库的secret:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ack-cr-push-secret
  annotations:
    tekton.dev/docker-0: https://registry.cn-hangzhou.aliyuncs.com
type: kubernetes.io/basic-auth
stringData:
  username: <cleartext non-encoded>
  password: <cleartext non-encoded>
```
执行如下命令：

```bash
kubectl apply -f tekton/pipeline/build-and-deploy-pipeline.yaml -f tekton/resources/picalc-git.yaml -f tekton/image-secret.yaml -f tekton/tasks/source-to-image.yaml -f tekton/tasks/image-to-deployer.yaml

```
### 部署 Knative Serving 服务
先创建deployer-github-trigger服务，用于接收GitHub事件，并触发Tekton Pipeline构建任务。其中 service.yaml 如下：

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: deployer-github-trigger
spec:
  template:
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/deployer-trigger:tekton-v1_74647e3a-20190806093544
        args:
          - --trigger-config=/app/config/deployer-trigger.yaml
        volumeMounts:
        - name: config-volume 
          mountPath: /app/config
      serviceAccountName: tekton
      volumes:
        - name: config-volume 
          configMap:
            name: deployer-trigger-config
            items:
              - key: deployer-trigger.yaml
                path: deployer-trigger.yaml
```
这里通过ConfigMap `deployer-trigger-config`， 设置 PipelineRun。deployer-github-trigger 能根据 github Event 信息获取代码仓库的最新信息但不能自动决定 PipelineRun 的定义，所以需要指定一个 PipelineRun 的模板。Trigger 通过 --trigger-config 参数指定 PipelineRun 的模板, 模板内容如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: deployer-trigger-config
  namespace: default
data:
  "deployer-trigger.yaml": |-
    apiVersion: tekton.dev/v1alpha1
    kind: PipelineRun
    metadata:
      name: tekton-kn-sample
    spec:
      pipelineRef:
        name: build-and-deploy-pipeline
      resources:
        - name: git-source
          resourceRef:
            name: eventing-tekton-serving-git
      params:
        - name: pathToContext
          value: "src"
        - name: pathToYamlFile
          value: ""
        - name: imageUrl
          value: "registry.cn-hangzhou.aliyuncs.com/knative-sample/eventing-tekton-serving-helloworld"
        - name: imageTag
          value: "1.0"
      serviceAccount: pipeline-account
```

执行命令如下：

```bash
kubectl apply -f serving/clusterrole.yaml -f serving/clusterrolebinding.yaml -f serving/serviceaccount.yaml -f serving/configmap.yaml -f serving/service.yaml
```

### 配置 Eventing 中 GitHub 事件源
代码merge request会触发对应的事件，通过Knative Eventing获取到事件之后直接将事件发送给deployer-github-trigger服务。

#### 创建 GitHub Token
创建 [Personal access tokens](https://github.com/settings/tokens), 用于访问 GitHub API。另外你的代码将使用它来验证来自github的传入webhook（secret token）。token 的名称可以任意设置。`Source` 需要开启 `repo:public_repo` 和 `admin:repo_hook` , 以便通过公共仓库触发 Event 事件，并为这些公共仓库创建 webhooks 。
下面是设置一个 "GitHubSource Sample" token 的示例。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1559396080243-c86eccce-8094-44d1-83a8-bf05dda974f1.png) 

secretToken内容可以通过下述方式生成随机字符串:

```bash
head -c 8 /dev/urandom | base64
```

更新 `githubsecret.yaml` 内容。如果生成的是 `personal_access_token_value` token, 则需要设置 `secretToken` 如下：

```yaml
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

```bash
kubectl  apply -f eventing/githubsecret.yaml
```
#### 创建 GitHub 事件源
为了接收 GitHub 产生的事件， 需要创建 GitHubSource 用于接收事件。

```yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: GitHubSource
metadata:
  name: deployer-github-sources
spec:
  eventTypes:
  - pull_request
  ownerAndRepository: knative-sample/eventing-tekton-serving
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
    name: deployer-github-trigger

```
关键字段解释：
  - 指定 github 仓库。ownerAndRepository: knative-sample/eventing-tekton-serving 表示监听 knative-sample/eventing-tekton-serving 仓库的事件
  - 事件类型
   eventTypes 是一个数组，这个数组中可以配置 github 事件列表
  - 认证信息。accessToken 和 secretToken 是通过 secret 引用 github 仓库的认证信息。
  - 目标 Service。sink 字段表示接收到的事件需要发送到哪个 Service , 这里是直接发送到前面定义的 deployer-github-trigger 服务

执行 kubectl 命令：
```bash
kubectl  apply -f eventing/github-source.yaml
```
如果集群中开启了Istio注入，需要开启egress访问：
```bash
kubectl  apply -f eventing/egress.yaml
```

`deployer-github-sources` 提交到 Kubernetes 之后，github source controller 会在 knative-sample/eventing-tekton-serving 下创建一下 webhook，回调地址就是我们的 github_receive_adapter 服务公网地址。
当 knative-sample/eventing-tekton-serving 有 pull request 发生时就会自动触发 deployer-github-trigger 的执行，deployer-github-trigger 首先编译镜像，然后更新 hello-sample service 镜像，从而完成自动化发布。

### 代码->镜像->服务
下面我们演示一下从代码到服务，自动化构建和部署过程：
![录制_2019_11_11_16_13_16_464.gif](https://intranetproxy.alipay.com/skylark/lark/0/2019/gif/11378/1573461046104-6b34e1ca-98c4-47e2-af4f-48ab0ebc648e.gif) 

服务访问体验地址：http://hello-sample.default.knative.kuberun.com

##  设置日志、监控告警
1. 选择**日志库**，创建 Logstore。这里以创建 helloworld 为例：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560392950566-4484bb5a-66f7-4800-a856-cc047d255eb0.png) 
2. 数据源接入，选择**Docker标准输出**， 参见[日志服务容器标准输出](https://help.aliyun.com/document_detail/66658.html)
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573044379880-abb6c8a1-e6fa-4ccc-bfb1-a5c34be6ab0f.png) 
3. 插件配置这里我们针对 `hello-sample` Service, 设置采集的环境变量为："K_SERVICE": "hello-sample"。并且通过 processors 分割日志信息，如这里"Keys": [ "time","level", "msg" ]。

```json
{
  "inputs": [
    {
      "detail": {
        "IncludeEnv": {
          "K_SERVICE": "hello-sample"
        },
        "IncludeLabel": {},
        "ExcludeLabel": {}
      },
      "type": "service_docker_stdout"
    }
  ],
  "processors": [
    {
      "detail": {
        "KeepSource": false,
        "NoMatchError": true,
        "Keys": [
          "time",
          "level",
          "msg"
        ],
        "NoKeyError": true,
        "Regex": "(\\d+-\\d+-\\d+\\s+\\d+:\\d+:\\d+)\\s+(\\w+)\\s+(.*)",
        "SourceKey": "content"
      },
      "type": "processor_regex"
    }
  ]
}
```
4. 设置查询分析，参考[查询与分析](https://help.aliyun.com/document_detail/53608.html)。为了便于分析这里设置 level、msg 和 time 这 3 列：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573044519740-29a9fc0b-761d-4ab0-b737-92c7b2e093a3.png) 
5. 访问 hello-sample 示例服务: http://hello-sample.default.knative.kuberun.com

6. 登录[日志服务控制台](https://sls.console.aliyun.com/), 进入对应的 Project, 选择 `helloworld` Logstore，点击**查询**，可以看到日志输出如图所示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573045607437-e3dec50e-e094-4a70-a859-928e03746edd.png) 

7. 设置查询 sql 语句。这里设置监控的原则是根据**ERROR**出现的次数，因此可以设计统计 ERROR 的 sql 语句：
```
* | select 'ERROR' , count(1) as total group by 'ERROR'
```
点击【查询/分析】，结果如图所示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573047329609-639b3f39-6d8c-4105-869f-31b21e77535e.png) 
8. 告警设置。点击 【另存为告警】。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573047364783-b3308ac5-6785-4f76-b55f-7be10c0e8db6.png) 
9. 设置告警名称、添加到仪表盘（这里可以新建，输入名称即可）等。其中告警触发条件输入判断告警是否触发的条件表达式, 可以参考[告警条件表达式语法](https://help.aliyun.com/document_detail/98379.html?spm=5176.2020520112.2.1.5eef34c0086IkI)。我们这里设置“查询区间：1 分钟，执行间隔：1 分组，触发条件：total > 3” 表示间隔 1 分钟检查，如果 1 分钟内出现3次 ERROR 信息，则触发告警。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560418522401-9333435a-b408-40f0-bec6-c66d86131d48.png) 
10. 告警通知
当前支持如图所示告警通知：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573094630665-d400b316-1930-468b-91be-60ebd39ae14c.png) 

11. 访问 hello-sample 示例服务。执行多次以下: http://hello-sample.default.knative.kuberun.com，就会触发告警通知

如果是设置的邮件通知，告警信息如下图所示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573094527725-7b144720-fea1-4627-96a6-10536995780b.png) 



## 设置分布式链路追踪
1. 选择命名空间设置如下标签启用 Sidecar 自动注入：istio-injection=enabled。通过这种方式就注入了 Istio 的 envoy 代理（proxy）容器， Istio 的 envoy 代理拦截流量后会主动上报 trace 系统。以设置 default 命名空间为例：

```
kubectl label namespace default istio-injection=enabled
```
2. 访问 hello-sample 示例服务: http://hello-sample.default.knative.kuberun.com
3. 登录[阿里云链路追踪服务控制台](https://tracing-analysis.console.aliyun.com/)， 选择**应用列表**，可以查看对应应用的 tracing 信息。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573096184024-83509d24-9cb2-4734-bd00-aeef4ef58b7b.png) 
4. 选择应用，点击查看**应用详情**，可以看到服务调用的平均响应时间。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573048268914-a9971504-44d3-43d2-9afb-94bd51a64686.png) 