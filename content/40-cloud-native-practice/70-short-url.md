---
title: "三步走！基于 Knative Serverless 技术实现一个短网址服务"
date: 2020-5-10T15:26:15Z
draft: false
weight: 70
description: ""
---

短网址顾名思义就是使用比较短的网址代替很长的网址。维基百科上面的解释是这样的：
> 短网址又称网址缩短、缩短网址、URL 缩短等，指的是一种互联网上的技术与服务，此服务可以提供一个非常短小的 URL 以代替原来的可能较长的URL，将长的 URL 位址缩短。用户访问缩短后的 URL 时通常将会重定向到原来的长 URL 

## 起源
虽然现在互联网已经非常发达了，但还是有很多场景会对用户输入的内容有长度限制。比如 ：
- 微薄、Twitter 长度不能超过 140 个字
- 一些早期的 BBS 文章单行的长度不能超过 78 字符等场景
- 运营商短信的长度不能超过 70 个字

而现在很多媒体、电商平台的内容大多都是多人协作通过比较复杂的系统、框架生成的，链接长度几十个甚至上百字符都是很平常的事情，所以如果在上述的几个场景中传播链接使用短网址服务就是一个必然的结果。比如下面这些短信截图你应该不会陌生：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1568212215442-ffb75ed1-e682-44b5-adb5-38d72ce8dc26.png) 

## 应用场景
短网址服务的最初本意就是缩短长 url，方便传播。但其实短网址服务还能做很多其他的事情。比如下面这些：

- 访问次数的限制，比如只能访问 1 次，第二次访问的时候就拒绝服务
- 时间的限制，比如只能在一周内提供访问服务，超过一周就拒绝服务
- 根据访问者的地域的限制
- 通过密码访问
-  访问量统计
-  高峰访问时间统计等等
- 统计访问者的一些信息，比如：
  - 来源城市
  - 访问时间
  - 使用的终端设备、浏览器
  - 访问来源 IP 
- 在营销活动中其实还可以对不同的渠道生成不通的短网址，这样通过统计这些短网址还能判断不同渠道的访问量等信息

## 基于 Knative Serverless 技术实现一个短网址服务
在 Knative 模式下可以实现按需分配，没有流量的时候实例缩容到零，当有流量进来的时候再自动扩容实例提供服务。
现在我们就基于阿里云容器服务的 Knative 来实现一个 serverless 模式的短网址服务。本示例会给出一个完整的 demo，你可以自己在阿里云容器服务上面创建一个 Knative 集群，使用本示例提供服务。本示例中实现一个最简单的功能
- 通过接口实现长网址到短网址的映射服务 
- 当用户通过浏览器访问短网址的时候通过 301 跳转到长网址

下面我们一步一步实现这个功能

### 数据库
既然要实现短网址到长网址的映射，那么就需要保存长网址的信息到数据库，并且生成一个短的 ID 作为短网址的一部分。所以我们首先需要选型使用什么数据库。在本示例中我们选择使用**阿里云的表格存储，表格存储最大的优势就是按量服务**，你只需要为你使用的量付费，而且价格也很实惠。如下所示的按量计费价格表。1G 的数据保存一年的费用是**3.65292元/年**( 0.000417 * 24 * 365=3.65292) ，是不是很划算。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1568260591334-552aed47-3e54-456a-aa2d-0a458f01b232.png) 

### 短网址生成 API
我们需要有一个 API 生成短网址

`/new?origin-url=${长网址}`

- origin-url 访问地址

**返回结果**
```
vEzm6v
```
假设我们服务的域名是 short-url.default.knative.kuberun.com ，那么现在访问 http://short-url.default.knative.kuberun.com/vEzm6v 就可以跳转到长网址了。
### 代码实现

```
package main

import (
	"crypto/md5"
	"encoding/hex"
	"fmt"
	"log"
	"net/http"
	"os"
	"strconv"
	"time"

	"strings"

	"github.com/aliyun/aliyun-tablestore-go-sdk/tablestore"
)

var (
	alphabet = []byte("abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ")
	l        = &Log{}
)

func ShortUrl(url string) string {
	md5Str := getMd5Str(url)
	var tempVal int64
	var result [4]string
	for i := 0; i < 4; i++ {
		tempSubStr := md5Str[i*8 : (i+1)*8]
		hexVal, _ := strconv.ParseInt(tempSubStr, 16, 64)
		tempVal = 0x3FFFFFFF & hexVal
		var index int64
		tempUri := []byte{}
		for i := 0; i < 6; i++ {
			index = 0x0000003D & tempVal
			tempUri = append(tempUri, alphabet[index])
			tempVal = tempVal >> 5
		}
		result[i] = string(tempUri)
	}
	return result[0]
}

func getMd5Str(str string) string {
	m := md5.New()
	m.Write([]byte(str))
	c := m.Sum(nil)
	return hex.EncodeToString(c)
}

type Log struct {
}

func (log *Log) Infof(format string, a ...interface{}) {
	log.log("INFO", format, a...)
}

func (log *Log) Info(msg string) {
	log.log("INFO", "%s", msg)
}

func (log *Log) Errorf(format string, a ...interface{}) {
	log.log("ERROR", format, a...)
}

func (log *Log) Error(msg string) {
	log.log("ERROR", "%s", msg)
}

func (log *Log) Fatalf(format string, a ...interface{}) {
	log.log("FATAL", format, a...)
}

func (log *Log) Fatal(msg string) {
	log.log("FATAL", "%s", msg)
}

func (log *Log) log(level, format string, a ...interface{}) {
	var cstSh, _ = time.LoadLocation("Asia/Shanghai")
	ft := fmt.Sprintf("%s %s %s\n", time.Now().In(cstSh).Format("2006-01-02 15:04:05"), level, format)
	fmt.Printf(ft, a...)
}

func handler(w http.ResponseWriter, r *http.Request) {
	l := &Log{}
	l.Infof("Hello world received a request, url: %s", r.URL.Path)
	l.Infof("url:%s ", r.URL)
	//if r.URL.Path == "/favicon.ico" {
	//	http.NotFound(w, r)
	//	return
	//}

	urls := strings.Split(r.URL.Path, "/")
	originUrl := getOriginUrl(urls[len(urls)-1])
	http.Redirect(w, r, originUrl, http.StatusMovedPermanently)
}

func new(w http.ResponseWriter, r *http.Request) {
	l.Infof("Hello world received a request, url: %s", r.URL)
	l.Infof("url:%s ", r.URL)
	originUrl, ok := r.URL.Query()["origin-url"]
	if !ok {
		l.Errorf("no origin-url params found")
		w.WriteHeader(http.StatusBadRequest)
		w.Write([]byte("Bad request!"))
		return
	}

	surl := ShortUrl(originUrl[0])
	save(surl, originUrl[0])
	fmt.Fprint(w, surl)

}

func getOriginUrl(surl string) string {
	endpoint := os.Getenv("OTS_TEST_ENDPOINT")
	tableName := os.Getenv("TABLE_NAME")
	instanceName := os.Getenv("OTS_TEST_INSTANCENAME")
	accessKeyId := os.Getenv("OTS_TEST_KEYID")
	accessKeySecret := os.Getenv("OTS_TEST_SECRET")
	client := tablestore.NewClient(endpoint, instanceName, accessKeyId, accessKeySecret)

	getRowRequest := &tablestore.GetRowRequest{}
	criteria := &tablestore.SingleRowQueryCriteria{}

	putPk := &tablestore.PrimaryKey{}
	putPk.AddPrimaryKeyColumn("id", surl)
	criteria.PrimaryKey = putPk

	getRowRequest.SingleRowQueryCriteria = criteria
	getRowRequest.SingleRowQueryCriteria.TableName = tableName
	getRowRequest.SingleRowQueryCriteria.MaxVersion = 1

	getResp, _ := client.GetRow(getRowRequest)
	colmap := getResp.GetColumnMap()
	return fmt.Sprintf("%s", colmap.Columns["originUrl"][0].Value)
}

func save(surl, originUrl string) {
	endpoint := os.Getenv("OTS_TEST_ENDPOINT")
	tableName := os.Getenv("TABLE_NAME")
	instanceName := os.Getenv("OTS_TEST_INSTANCENAME")
	accessKeyId := os.Getenv("OTS_TEST_KEYID")
	accessKeySecret := os.Getenv("OTS_TEST_SECRET")
	client := tablestore.NewClient(endpoint, instanceName, accessKeyId, accessKeySecret)

	putRowRequest := &tablestore.PutRowRequest{}
	putRowChange := &tablestore.PutRowChange{}
	putRowChange.TableName = tableName

	putPk := &tablestore.PrimaryKey{}
	putPk.AddPrimaryKeyColumn("id", surl)
	putRowChange.PrimaryKey = putPk

	putRowChange.AddColumn("originUrl", originUrl)
	putRowChange.SetCondition(tablestore.RowExistenceExpectation_IGNORE)
	putRowRequest.PutRowChange = putRowChange

	if _, err := client.PutRow(putRowRequest); err != nil {
		l.Errorf("putrow failed with error: %s", err)
	}
}

func main() {
	http.HandleFunc("/", handler)
	http.HandleFunc("/new", new)
	port := os.Getenv("PORT")
	if port == "" {
		port = "9090"
	}

	if err := http.ListenAndServe(fmt.Sprintf(":%s", port), nil); err != nil {
		log.Fatalf("ListenAndServe error:%s ", err.Error())
	}

}

```
代码我已经编译成镜像，你可以直接使用 registry.cn-hangzhou.aliyuncs.com/knative-sample/shorturl:v1 此镜像编部署服务。

## 三步走起！！！
### 第一步 准备数据库
首先到[阿里云开通表格存储服务](https://www.aliyun.com/product/ots?spm=5176.12825654.eofdhaal5.65.e9392c4aopAdh5&aly_as=i2h7dcdQ)，然后创建一个实例和表。我们需要的结构比较简单，只需要短 URL ID 到长 URL 的映射即可，存储表结构设计如下：

| 名称      | 描述      |
| --------- | --------- |
| id        | 短网址 ID |
| originUrl | 长网址    |

### 第二步 获取 access key
登陆到阿里云以后鼠标浮动在页面的右上角头像，然后点击 accesskeys 跳转到 accesskeys 管理页面
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1568965254325-07b61d33-f453-41f2-ab7b-06d63d4bed37.png) 
点击显示即可显示 Access Key Secret
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1568965283426-9951452a-7023-4073-b8a2-f493e834db34.png) 
### 第三步 部署服务
Knative Service 的配置如下， 使用前两步的配置信息填充 Knative Service 的环境变量。然后部署到 Knative集群即可
```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: short-url
  namespace: default
spec:
  template:
    metadata:
      labels:
        app: short-url
      annotations:
        autoscaling.knative.dev/maxScale: "20"
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/target: "100"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/shorturl:v1
          ports:
            - name: http1
              containerPort: 8080
          env:
          - name: OTS_TEST_ENDPOINT
            value: http://t.cn-hangzhou.ots.aliyuncs.com
          - name: TABLE_NAME
            value: ${TABLE_NAME}
          - name: OTS_TEST_INSTANCENAME
            value: ${OTS_TEST_INSTANCENAME}
          - name: OTS_TEST_KEYID
            value: ${OTS_TEST_KEYID}
          - name: OTS_TEST_SECRET
            value: ${OTS_TEST_SECRET}
```
把上面的这个 service 中 ${TABLE_NAME} 这些变量替换成具体的值，然后创建 knative service：
```
└─# kubectl get ksvc
short-url       http://short-url.default.knative.kuberun.com       short-url-456q9       short-url-456q9       True
```
现在可以开始测试
- 生成一个短网址
```
└─# curl 'http://short-url.default.knative.kuberun.com/new?origin-url=https://help.aliyun.com/document_detail/121534.html?spm=a2c4g.11186623.6.786.41e074d9oHpbO2'
vEzm6v
```
curl 命令输出的结果 VR7baa 就是短网址的 ID
- 访问短网址
在浏览器中打开 [http://short-url.default.knative.kuberun.com/vEzm6v](http://short-url.default.knative.kuberun.com/vEzm6v) 就能跳转到长 url 网址了。

## 小结
本实战我们只需三步就基于 Knative 实现了一个 Serverless 的短网址服务，此短网址服务在没有请求的时候可以缩容到零节省计算资源，在有很多请求的时候可以自动扩容。并且使用了阿里云表格存储，这样数据库也是按需付费。基于 Knative + TableStore 实现了短网址服务的 Serverless 化。


