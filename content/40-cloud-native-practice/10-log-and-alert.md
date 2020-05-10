---
title: "日志和监控告警"
date: 2020-5-10T15:26:15Z
draft: false
weight: 10
description: ""
---

阿里云日志服务（Log Service，简称 LOG）是针对日志类数据的一站式服务。您无需开发就能快捷完成日志数据采集、消费、投递、查询分析以及监控告警等功能。在 Knative 中结合日志服务，能有效提升对 Serverless 应用的运维能力。
## 前提条件
- 您已经成功创建一个 Kubernetes 集群，参见[创建Kubernetes集群](https://help.aliyun.com/document_detail/86488.html?spm=a2c4g.11186623.2.10.571b21d6Lo30j5#CS-user-guide-kubernetes)
- 部署[日志服务](https://help.aliyun.com/document_detail/87540.html)
- 部署成功 Knative , 参见[部署Knative](https://help.aliyun.com/document_detail/121509.html)
- 部署成功 Serving 组件

## 日志
1. 部署 Knative Service 服务。参考[部署 Serving Hello World 应用示例](https://help.aliyun.com/document_detail/121534.html)
2. 选择**日志库**，创建Logstore。这里以创建helloworld为例：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560392950566-4484bb5a-66f7-4800-a856-cc047d255eb0.png) 
3. 数据源接入，选择**Docker标准输出**， 参见[日志服务容器标准输出](https://help.aliyun.com/document_detail/66658.html)
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573044379880-abb6c8a1-e6fa-4ccc-bfb1-a5c34be6ab0f.png) 
4. 插件配置这里我们针对 `helloworld-go` Service, 设置采集的环境变量为："K_SERVICE": "helloworld-go"。并且通过 processors 分割日志信息，如这里"Keys": [ "time","level", "msg" ]。

```
{
  "inputs": [
    {
      "detail": {
        "IncludeEnv": {
          "K_SERVICE": "helloworld-go"
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
4. 开启全文索引，设置查询展示列
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560418766446-d91e0ab5-3103-457e-bf82-cbfc7dcc79fd.png) 
5. 访问 Hello World 示例服务。

```
$ curl -H "Host: helloworld-go.default.example.com" http://112.124.XX.XX
Hello Go Sample v1!
```
6. 登录[日志服务控制台](https://sls.console.aliyun.com/), 进入对应的 Project, 选择 `helloworld` Logstore，点击**查询**
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573095922742-2e991fce-5641-4f65-b4e9-fa0cadb31a21.png) 
## 监控告警
1. 设置查询分析，参考[查询与分析](https://help.aliyun.com/document_detail/53608.html)
- 为了便于查看，可以通过**列设置**，显示所需要的列， 这里设置 level、msg 和 time 这 3 列：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573095854993-254655ba-5160-443a-943d-0683cd1243e5.png) 
- 设置查询sql语句。这里我们监控的原则是根据**ERROR**出现的次数，因此可以设计统计 ERROR 的sql语句：
```
* | select 'ERROR' , count(1) as total group by 'ERROR'
```
点击【查询/分析】，结果如图所示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560415510839-c07e2e34-efa4-47d7-85d4-ed05192016f1.png) 
2. 告警设置。点击 【另存为告警】。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1560417912609-a2fb52d5-a4a0-43a8-a3ca-e2853ae651dc.png) 
3. 设置告警名称、添加到仪表盘（这里可以新建，输入名称即可）等。其中告警触发条件输入判断告警是否触发的条件表达式, 可以参考[告警条件表达式语法](https://help.aliyun.com/document_detail/98379.html?spm=5176.2020520112.2.1.5eef34c0086IkI)。我们这里设置“查询区间：1 分钟，执行间隔：1 分组，触发条件：total > 3” 表示间隔 1 分钟检查，如果 1 分钟内出现3次 ERROR 信息，则触发告警。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573093737132-f30cdf3e-0e13-4e97-a5c6-8e3813685f31.png) 
4. 告警通知
当前支持如图所示告警通知：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573094671855-a327a3ad-dfec-46a1-a109-3e01eb327453.png) 

5. 访问 Hello World 示例服务。执行多次以下命令，就会触发告警通知

```
$ curl -H "Host: helloworld-go.default.example.com" http://112.124.XX.XX
Hello Go Sample v1!
```
如果是设置的邮件通知，告警信息如下图所示：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11378/1573095719186-2a40ec0d-498e-408c-a3c6-5739faeba157.png) 
## 总结
通过上面的介绍相信你已经了解如何在 Knative 中使用日志服务收集 Serverless 应用容器日志，并进行告警设置。在 Knative 中采用日志服务收集、分析业务日志，并结合设置监控告警，满足了生产级别的 Serverless 应用运维的诉求。
