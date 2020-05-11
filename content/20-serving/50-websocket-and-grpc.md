---
title: "WebSocket 和 gRPC 服务"
date: 2020-5-10T15:26:15Z
draft: false
weight: 50
description: "calling built-in Shortcodes into your content files."
---

虽然说 Knative 默认就支持 WebSocket 和 gRPC，但在使用中发现有些时候想要把自己的 WebSocket 或  gRPC 部署到 Knative 中还是会有各种不顺利的地方，尽管最后排查发现大多都是自己的程序问题或者是配置错误导致的。为了方便大家做验证，这里就分别给出一个 WebSocket 的例子和一个 gRPC 的例子。当我们需要在生产或者测试环境部署相关服务的时候可以使用本文给出的示例进行 Knative 服务的测试。

## WebSocket 
如果自己手动的配置 Istio Gateway 支持 WebSocket 就需要开启 `websocketUpgrade` 功能。但使用 Knative Serving 部署其实就自带了这个能力。本示例的完整代码放在 https://github.com/knative-sample/websocket-chat/tree/b1.0 ，这是一个基于 WebSocket 实现的群聊的例子。使用浏览器连接到部署的服务中就可以看到一个接收信息的窗口和发送信息的窗口。当你发出一条信息以后所有连接进来的用户都能收到你的消息。所以你可以使用两个浏览器窗口分别连接到服务中，一个窗口发送消息一个窗口接收消息，以此来验证 WebSocket 服务是否正常。

本示例是在 [gorilla/websocket](https://github.com/gorilla/websocket/tree/master/examples/chat)  基础之上进行了一些优化:
- 代码中添加了 vendor 依赖，你下载下来就可以直接使用
- 添加了 Dockerfile 和 Makefile 可以直接编译二进制和制作镜像
- 添加了 Knative Sevice 的 yaml 文件(`service.yaml`)，你可以直接提交到 Knative 集群中使用
- 也可以直接使用编译好的镜像 `registry.cn-hangzhou.aliyuncs.com/knative-sample/websocket-chat:2019-10-15`

Knative Service 配置
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: websocket-chat
spec:
  template:
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/websocket-chat:2019-10-15
          ports:
            - name: http1
              containerPort: 8080
```

代码 clone 下来以后执行 `kubectl apply -f service.yaml ` 把服务部署到 Knative 中，然后直接访问服务地址即可使用。
查看 ksvc 列表，并获取访问域名

```bash
└─# kubectl get ksvc
NAME                            URL                                                                LATESTCREATED                         LATESTREADY                           READY   REASON
websocket-chat                  http://websocket-chat.default.knative.kuberun.com                  websocket-chat-tp4ph                  websocket-chat-tp4ph                  True

```
现在使用浏览器打开 [http://websocket-chat.default.knative.kuberun.com](http://websocket-chat.default.knative.kuberun.com) 即可看到群聊窗口

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/11431/1573019049885-8a06db87-768d-4502-a81a-9a2048979a72.png) 
打开两个窗口，在其中一个窗口发送一条消息，另外一个窗口通过 WebSocket 也收到了这条消息。

## gRPC 
gRPC 不能通过浏览器直接访问，需要通过 client 端和 server 端进行交互。本示例的完整代码放在 https://github.com/knative-sample/grpc-ping-go/tree/b1.0 ，本示例会给一个可以直接使用的镜像，测试 gRPC 服务。

Knative Service 配置
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: grpc-ping
spec:
  template:
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/grpc-ping-go:2019-10-15
        ports:
          - name: h2c
            containerPort: 8080

```
代码 clone 下来以后执行 `kubectl apply -f service.yaml ` 把服务部署到 Knative 中。
获取 ksvc 列表和访问域名：
```bash
└─# kubectl get ksvc
NAME                            URL                                                                LATESTCREATED                         LATESTREADY                           READY     REASON
grpc-ping                       http://grpc-ping.default.knative.kuberun.com                       grpc-ping-plzrv                       grpc-ping-plzrv                       True
```

现在我们已经知道 gRPC  server 的地址是 grpc-ping.default.knative.kuberun.com  端口是 80 那么我们可以发起测试请求：

```bash
└─# docker run --rm registry.cn-hangzhou.aliyuncs.com/knative-sample/grpc-ping-go:2019-10-15 /client -server_addr="grpc-ping.default.knative.kuberun.com:80" -insecure
2019/11/06 13:45:46 Ping got hello - pong
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.43385349 +0800 CST m=+52.762218971
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433885075 +0800 CST m=+52.762250507
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433894386 +0800 CST m=+52.762259840
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433902205 +0800 CST m=+52.762267652
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433909964 +0800 CST m=+52.762275418
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433926773 +0800 CST m=+52.762292207
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.43393548 +0800 CST m=+52.762300916
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433940721 +0800 CST m=+52.762306150
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433954408 +0800 CST m=+52.762319841
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433959768 +0800 CST m=+52.762325212
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433964951 +0800 CST m=+52.762330381
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433973361 +0800 CST m=+52.762338796
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433986818 +0800 CST m=+52.762352250
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.433994339 +0800 CST m=+52.762359790
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.434001938 +0800 CST m=+52.762367370
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.434016244 +0800 CST m=+52.762381708
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.434024768 +0800 CST m=+52.762390227
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.434037418 +0800 CST m=+52.762402862
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.434045691 +0800 CST m=+52.762411130
2019/11/06 13:45:46 Got pong 2019-11-06 13:45:46.434053459 +0800 CST m=+52.762418894
```

## 小结
本文通过两个例子分别展示了 WebSocket 和 gRPC 的部署方法：
- WebSocket 示例通过一个 chat 的方式展示发送和接受消息
- gRPC 通过启动一个 client 的方式展示 gRPC 远程调用的过程

## 参考资料
- WebSocket 示例代码 https://github.com/knative-sample/websocket-chat/tree/b1.0 
- gRPC 示例代码 https://github.com/knative-sample/grpc-ping-go/tree/b1.0


