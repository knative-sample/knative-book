---
title: "基于 Knative Serverless 技术实现天气服务"
date: 2020-5-10T15:26:15Z
draft: false
weight: 80
description: ""
---

提到天气预报服务，我们第一反应是很简单的一个服务啊，目前网上有大把的天气预报 API 可以直接使用，有必要去使用 Knative 搞一套吗？杀鸡用牛刀？先不要着急，我们先看一下实际的几个场景需求：
- 场景需求1：根据当地历年的天气信息，预测明年大致的高温到来的时间
- 场景需求2：近来天气多变，如果明天下雨，能否在早上上班前，给我一个带伞提醒通知
- 场景需求3：领导发话：最近经济不景气，公司财务紧张，那个服务器，你们提供天气、路况等服务的那几个小程序一起用吧，但要保证正常提供服务。

从上面的需求，我们其实发现，要做好一个天气预报的服务，也面临内忧（资源紧缺）外患（需求增加），并不是那么简单的。不过现在更不要着急，我们可以使用 Knative 帮你解决上面的问题。
 
**关键词**：天气查询、表格存储，通道服务，事件通知
## 场景需求
首先我们来描述一下我们要做的天气服务场景需求：
### 1. 提供对外的天气预报 RESTful API
- 根据城市、日期查询（支持未来 3 天）国内城市天气信息
- 不限制查询次数，支持较大并发查询（1000）
### 2. 天气订阅提醒
- 订阅国内城市天气信息，根据实际订阅城市区域，提醒明天下雨带伞
- 使用钉钉进行通知

## 整体架构
有了需求，那我们就开始如何基于 Knative 实现天气服务。我们先看一下整体架构：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1569378819184-451c50bd-dd3d-4ddd-9b9e-9f33fdbd749a.png) 

- 通过 CronJob 事件源，每隔 3个 小时定时发送定时事件，将国内城市未来3天的天气信息，存储更新到表格存储
- 提供 RESTful API 查询天气信息
- 通过表格存储提供的通道服务，实现 TableStore 事件源
- 通过 Borker/Trigger 事件驱动模型，订阅目标城市天气信息
- 根据订阅收到的天气信息进行钉钉消息通知。如明天下雨，提示带伞等

## 提供对外的天气预报 RESTful API
### 对接高德开放平台天气预报 API
查询天气的 API 有很多，这里我们选择高德开放平台提供的天气查询 API，使用简单、服务稳定，并且该天气预报 API 每天提供 100000 免费的调用量，支持国内 3500 多个区域的天气信息查询。另外高德开放平台，除了天气预报，还可以提供 ip 定位、搜索服务、路径规划等，感兴趣的也可以研究一下玩法。
登录高德开放平台: https://lbs.amap.com， 创建应用，获取 Key 即可：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1569331307691-e863888e-0be0-451a-bbaa-fe06fb3f61d2.png) 
获取Key之后，可以直接通过url访问：https://restapi.amap.com/v3/weather/weatherInfo?city=110101&extensions=all&key=<用户key>，返回天气信息数据如下：
```json
{
    "status":"1",
    "count":"1",
    "info":"OK",
    "infocode":"10000",
    "forecasts":[
        {
            "city":"杭州市",
            "adcode":"330100",
            "province":"浙江",
            "reporttime":"2019-09-24 20:49:27",
            "casts":[
                {
                    "date":"2019-09-24",
                    "week":"2",
                    "dayweather":"晴",
                    "nightweather":"多云",
                    "daytemp":"29",
                    "nighttemp":"17",
                    "daywind":"无风向",
                    "nightwind":"无风向",
                    "daypower":"≤3",
                    "nightpower":"≤3"
                },
                ...
            ]
        }
    ]
}
```

### 定时同步并更新天气信息
#### 同步并更新天气信息
该功能主要实现对接高德开放平台天气预报 API， 获取天气预报信息，同时对接阿里云表格存储服务（TableStore），用于天气预报数据存储。具体操作如下：
- 接收 CloudEvent 定时事件
- 查询各个区域天气信息
- 将天气信息存储或者更新到表格存储

在 Knative 中，我们可以直接创建服务如下：
```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: weather-store
  namespace: default
spec:
  template:
    metadata:
      labels:
        app: weather-store
      annotations:
        autoscaling.knative.dev/maxScale: "20"
        autoscaling.knative.dev/target: "100"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/weather-store:1.2
          ports:
            - name: http1
              containerPort: 8080
          env:
          - name: OTS_TEST_ENDPOINT
            value: http://xxx.cn-hangzhou.ots.aliyuncs.com
          - name: TABLE_NAME
            value: weather
          - name: OTS_TEST_INSTANCENAME
            value: ${xxx} 
          - name: OTS_TEST_KEYID
            value: ${yyy}
          - name: OTS_TEST_SECRET
            value: ${Pxxx}
          - name: WEATHER_API_KEY
            value: xxx
```

关于服务具体实现参见 GitHub 源代码：https://github.com/knative-sample/weather-store
#### 创建定时事件
这里或许有疑问：为什么不在服务中直接进行定时轮询，非要通过 Knative Eventing 搞一个定时事件触发执行调用？那我们要说明一下，Serverless 时代下就该这样玩-按需使用。千万不要在服务中按照传统的方式空跑这些定时任务，亲，这是在持续浪费计算资源。
言归正传，下面我们使用 Knative Eventing 自带的定时任务数据源（CronJobSource），触发定时同步事件。
创建 CronJobSource 资源，实现每 3 个小时定时触发同步天气服务（weather-store），WeatherCronJob.yaml 如下：
```yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CronJobSource
metadata:
  name: weather-cronjob
spec:
  schedule: "0 */3 * * *"
  data: '{"message": "sync"}'
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: weather-store
```
执行命令：

```bash
kubectl apply -f WeatherCronJob.yaml
```
现在我们登录阿里云表格存储服务，可以看到天气预报数据已经按照城市、日期的格式同步进来了。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1569394615367-fcaf14c8-a779-4ede-88e4-3375b86d79b1.png) 

### 提供天气预报查询 RESTful API
有了这些天气数据，可以随心所欲的提供属于我们自己的天气预报服务了（感觉像是承包了一块地，我们来当地主），这里没什么难点，从表格存储中查询对应的天气数据，按照返回的数据格式进行封装即可。
在 Knative 中，我们可以部署 RESTful API 服务如下：

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: weather-service
  namespace: default
spec:
  template:
    metadata:
      labels:
        app: weather-service
      annotations:
        autoscaling.knative.dev/maxScale: "20"
        autoscaling.knative.dev/target: "100"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/weather-service:1.1
          ports:
            - name: http1
              containerPort: 8080
          env:
          - name: OTS_TEST_ENDPOINT
            value: http://xxx.cn-hangzhou.ots.aliyuncs.com
          - name: TABLE_NAME
            value: weather
          - name: OTS_TEST_INSTANCENAME
            value: ${xxx} 
          - name: OTS_TEST_KEYID
            value: ${yyy}
          - name: OTS_TEST_SECRET
            value: ${Pxxx}
```
具体实现源代码 GitHub 地址：https://github.com/knative-sample/weather-service
查询天气 RESTful API:
- 请求URL
GET /api/weather/query
```
参数:
cityCode：城市区域代码。如北京市区域代码：110000
date：查询日期。如格式：2019-09-26
```
- 返回结果
```json
{
    "code":200,
    "message":"",
    "data":{
        "adcode":"110000",
        "city":"北京市",
        "date":"2019-09-26",
        "daypower":"≤3",
        "daytemp":"30",
        "dayweather":"晴",
        "daywind":"东南",
        "nightpower":"≤3",
        "nighttemp":"15",
        "nightweather":"晴",
        "nightwind":"东南",
        "province":"北京",
        "reporttime":"2019-09-25 14:50:46",
        "week":"4"
    }
}
```
查询：杭州，2019-09-26天气预报信息示例
测试地址：http://weather-service.default.knative.kuberun.com/api/weather/query?cityCode=330100&date=2019-11-06
另外城市区域代码表可以在上面提供的源代码 GitHub 中可以查看，也可以到高德开放平台中下载：https://lbs.amap.com/api/webservice/download

## 天气订阅提醒
首先我们介绍一下表格存储提供的通道服务。通道服务（Tunnel Service）是基于表格存储数据接口之上的全增量一体化服务。通道服务为您提供了增量、全量、增量加全量三种类型的分布式数据实时消费通道。通过为数据表建立数据通道，您可以简单地实现对表中历史存量和新增数据的消费处理。通过数据通道可以进行数据同步、事件驱动、流式数据处理以及数据搬迁。这里事件驱动正好契合我们的场景。

### 自定义 TableStore 事件源
在 Knative  中自定义事件源其实很容易，可以参考官方提供的自定义事件源的实例：https://github.com/knative/docs/tree/master/docs/eventing/samples/writing-a-source。
我们这里定义数据源为 AliTablestoreSource。代码实现主要分为两部分：
1. 资源控制器-Controller：接收 AliTablestoreSource 资源，在通道服务中创建 Tunnel。
2. 事件接收器-Receiver：通过 Tunnel Client 监听事件，并将接收到的事件发送到目标服务( Broker）

关于自定义 TableStore 事件源实现参见 GitHub 源代码：https://github.com/knative-sample/tablestore-source

部署自定义事件源服务如下：
从 https://github.com/knative-sample/tablestore-source/tree/master/config 中可以获取事件源部署文件，执行下面的操作
```bash
 kubectl apply -f 200-serviceaccount.yaml -f 201-clusterrole.yaml -f 202-clusterrolebinding.yaml -f 300-alitablestoresource.yaml -f 400-controller-service.yaml -f 500-controller.yaml -f 600-istioegress.yaml
```
部署完成之后，我们可以看资源控制器已经开始运行：
```bash
[root@iZ8vb5wa3qv1gwrgb3lxqpZ config]# kubectl -n knative-sources get pods
NAME                                 READY   STATUS    RESTARTS   AGE
alitablestore-controller-manager-0   1/1     Running   0          4h12m
```

###  创建事件源
由于我们是通过 Knative Eventing 中 Broker/Trigger 事件驱动模型对天气事件进行处理。首先我们创建用于数据接收的 Broker 服务。
#### 创建 Broker
```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: Broker
metadata:
  name: weather
spec:
  channelTemplateSpec:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
```
#### 创建事件源实例
这里需要说明一下，创建事件源实例其实就是在表格存储中创建通道服务，那么就需要配置访问通道服务的地址、accessKeyId和accessKeySecret，这里参照格式：`{ "url":"https://xxx.cn-beijing.ots.aliyuncs.com/", "accessKeyId":"xxxx","accessKeySecret":"xxxx" }` 设置并进行base64编码。将结果设置到如下 Secret 配置文件`alitablestore` 属性中：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alitablestore-secret
type: Opaque
data:
  # { "url":"https://xxx.cn-beijing.ots.aliyuncs.com/", "accessKeyId":"xxxx","accessKeySecret":"xxxx" }
  alitablestore: "<base64>"
```
创建 RBAC 权限
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eventing-sources-alitablestore
subjects:
- kind: ServiceAccount
  name: alitablestore-sa
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: eventing-sources-alitablestore-controller

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alitablestore-sa
secrets:
- name: alitablestore-secret
```
创建 AliTablestoreSource 实例，这里我们设置接收事件的 `sink` 为上面创建的 Broker 服务。
```yaml
---
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: AliTablestoreSource
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: alitablestoresource
spec:
  # Add fields here
  serviceAccountName: alitablestore-sa
  accessToken:
    secretKeyRef:
      name: alitablestore-secret
      key: alitablestore
  tableName: weather
  instance: knative-weather
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Broker
    name: weather
```
创建完成之后，我们可以看到运行中的事件源：

```bash
[root@iZ8vb5wa3qv1gwrgb3lxqpZ config]# kubectl get pods
NAME                                                              READY   STATUS      RESTARTS   AGE
tablestore-alitablestoresource-9sjqx-656c5bf84b-pbhvw             1/1     Running     0          4h9m
```
### 订阅事件和通知提醒
#### 创建天气提醒服务
如何进行钉钉通知呢，我们可以创建一个钉钉的群组（可以把家里人组成一个钉钉群，天气异常时，给家人一个提醒），添加群机器人:
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1570696434119-788b81cf-632b-4cbe-b88d-c8341101a8b0.png) 
获取 webhook :
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1570697515570-49b009a0-dbfd-4be4-a80c-c14479d95108.png) 

这里我们假设北京(110000)，日期：2019-10-13, 如果天气有雨，就通过钉钉发送通知提醒，则服务配置如下：
```yaml
apiVersion: serving.knative.dev/v1beta1
kind: Service
metadata:
  name: day-weather
spec:
  template:
    spec:
      containers:
      - args:
        - --dingtalkurl=https://oapi.dingtalk.com/robot/send?access_token=xxxxxx
        - --adcode=110000
        - --date=2019-10-13
        - --dayweather=雨
        image: registry.cn-hangzhou.aliyuncs.com/knative-sample/dingtalk-weather-service:1.2
```
关于钉钉提醒服务具体实现参见 GitHub 源代码：https://github.com/knative-sample/dingtalk-weather-service
#### 创建订阅
最后我们创建 Trigger订阅天气事件，并且触发天气提醒服务：
```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: weather-trigger
spec:
  broker: weather
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: day-weather
```

订阅之后，如果北京(110000)，日期：2019-10-13, 天气有雨，会收到如下的钉钉提醒：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1570697061949-deacdc91-6f0e-4db1-b4d1-667639f610d2.png) 

这里其实还有待完善的地方：
- 是否可以基于城市进行订阅（只订阅目标城市）？
- 是否可以指定时间发送消息提醒（当天晚上 8 点准时推送第 2 天的天气提醒信息）？

有兴趣的可以继续完善当前的天气服务功能。

## 总结
通过上面的介绍，大家对如何通过 Knative 提供天气查询、 订阅天气信息，钉钉推送通知提醒应该有了更多的体感，其实类似的场景我们有理由相信通过 Knative Serverless 可以帮你做到资源利用游刃有余。欢迎持续关注。