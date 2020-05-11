---
title: "Parallel 解析"
date: 2020-5-10T15:26:15Z
draft: false
weight: 1000
description: "calling built-in Shortcodes into your content files."
---

从 Knative Eventing 0.8 开始，支持根据不同的过滤条件对事件进行选择处理。通过 Parallel 提供了这样的能力。本文就给大家介绍一下这个特性。

## 资源定义
我们先看一下 Parallel 资源定义，典型的 Parallel Spec描述如下：

```yaml
apiVersion: messaging.knative.dev/v1alpha1
kind: Parallel
metadata:
  name: me-odd-even-parallel
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
  cases:
    - filter:
        uri: "http://me-even-odd-switcher.default.svc.cluster.local/0"
      subscriber:
        ref:
          apiVersion: serving.knative.dev/v1alpha1
          kind: Service
          name: me-even-transformer
    - filter:
        uri: "http://me-even-odd-switcher.default.svc.cluster.local/1"
      subscriber:
        ref:
          apiVersion: serving.knative.dev/v1alpha1
          kind: Service
          name: me-odd-transformer
  reply:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: me-event-display
```
主要包括如下 3 部分：
- `cases` 定义了一系列 filter 和 subscriber。对于每个条件分支：
   - 首先判断 `filter`， 当返回事件时，调用 subscriber。filter和subscriber要求都是可访问的。
   - subscriber 执行返回的事件会发生到 reply。如果 reply 为空，则发送到 `spec.reply`
- `channelTemplate` 定义了当前 Parallel 中使用的Channel类型
- `reply` 定义了全局响应的目标函数。

逻辑架构如图所示：
![image](https://yqfile.alicdn.com/7ebd0f3731ec8a2f95a3c4ea9f33a4abbf1d4579.png)
 
## 代码实现
关键代码实现如下：
1. 首先为 Parallel 创建一个全局的 Channel。然后为每一个`case`创建一个过滤 Channel
2. 在每个`case`中做了如下处理：
   - 为全局的 Channel创建一个 Subscription，订阅条件为`filter`信息，并且把 reply 响应发送给当前 `case`中的过滤 Channel
   - 为过滤 Channel 创建一个 Subscription，将订阅信息发送给每个`case`中的 `Reply`。如果当前`case`中没有设置`Reply`，则发送的全局`Reply`。

```go
func (r *Reconciler) reconcile(ctx context.Context, p *v1alpha1.Parallel) error {
	p.Status.InitializeConditions()

	// Reconciling parallel is pretty straightforward, it does the following things:
	// 1. Create a channel fronting the whole parallel and one filter channel per branch.
	// 2. For each of the Branches:
	//     2.1 create a Subscription to the fronting Channel, subscribe the filter and send reply to the filter Channel
	//     2.2 create a Subscription to the filter Channel, subcribe the subscriber and send reply to
	//         either the branch Reply. If not present, send reply to the global Reply. If not present, do not send reply.
	// 3. Rinse and repeat step #2 above for each branch in the list
	if p.DeletionTimestamp != nil {
		// Everything is cleaned up by the garbage collector.
		return nil
	}

	channelResourceInterface := r.DynamicClientSet.Resource(duckroot.KindToResource(p.Spec.ChannelTemplate.GetObjectKind().GroupVersionKind())).Namespace(p.Namespace)

	if channelResourceInterface == nil {
		msg := fmt.Sprintf("Unable to create dynamic client for: %+v", p.Spec.ChannelTemplate)
		logging.FromContext(ctx).Error(msg)
		return errors.New(msg)
	}

	// Tell tracker to reconcile this Parallel whenever my channels change.
	track := r.resourceTracker.TrackInNamespace(p)

	var ingressChannel *duckv1alpha1.Channelable
	channels := make([]*duckv1alpha1.Channelable, 0, len(p.Spec.Branches))
	for i := -1; i < len(p.Spec.Branches); i++ {
		var channelName string
		if i == -1 {
			channelName = resources.ParallelChannelName(p.Name)
		} else {
			channelName = resources.ParallelBranchChannelName(p.Name, i)
		}

		c, err := r.reconcileChannel(ctx, channelName, channelResourceInterface, p)
		if err != nil {
			logging.FromContext(ctx).Error(fmt.Sprintf("Failed to reconcile Channel Object: %s/%s", p.Namespace, channelName), zap.Error(err))
			return err

		}
		// Convert to Channel duck so that we can treat all Channels the same.
		channelable := &duckv1alpha1.Channelable{}
		err = duckapis.FromUnstructured(c, channelable)
		if err != nil {
			logging.FromContext(ctx).Error(fmt.Sprintf("Failed to convert to Channelable Object: %s/%s", p.Namespace, channelName), zap.Error(err))
			return err

		}
		// Track channels and enqueue parallel when they change.
		if err = track(utils.ObjectRef(channelable, channelable.GroupVersionKind())); err != nil {
			logging.FromContext(ctx).Error("Unable to track changes to Channel", zap.Error(err))
			return err
		}
		logging.FromContext(ctx).Info(fmt.Sprintf("Reconciled Channel Object: %s/%s %+v", p.Namespace, channelName, c))

		if i == -1 {
			ingressChannel = channelable
		} else {
			channels = append(channels, channelable)
		}
	}
	p.Status.PropagateChannelStatuses(ingressChannel, channels)

	filterSubs := make([]*v1alpha1.Subscription, 0, len(p.Spec.Branches))
	subs := make([]*v1alpha1.Subscription, 0, len(p.Spec.Branches))
	for i := 0; i < len(p.Spec.Branches); i++ {
		filterSub, sub, err := r.reconcileBranch(ctx, i, p)
		if err != nil {
			return fmt.Errorf("Failed to reconcile Subscription Objects for branch: %d : %s", i, err)
		}
		subs = append(subs, sub)
		filterSubs = append(filterSubs, filterSub)
		logging.FromContext(ctx).Debug(fmt.Sprintf("Reconciled Subscription Objects for branch: %d: %+v, %+v", i, filterSub, sub))
	}
	p.Status.PropagateSubscriptionStatuses(filterSubs, subs)

	return nil
}
```

## 示例演示
接下来让我们通过一个实例具体了解一下 Parallel 。通过CronJobSource产生事件发送给 `me-odd-even-parallel` Parallel,  Parallel 会将事件发送给每个`case`， Case中通过 filter 不同的参数访问` me-even-odd-switcher`服务，  ` me-even-odd-switcher`服务会根据当前事件的创建时间随机计算0或1的值，如果计算值和请求参数值相匹配，则返回事件，否则不返回事件。
- 若`http://me-even-odd-switcher.default.svc.cluster.local/0`匹配成功，返回事件到`me-even-transformer`服务进行处理
- 若`http://me-even-odd-switcher.default.svc.cluster.local/1`匹配成功，返回事件到`odd-transformer`服务进行处理

不管哪个`case`处理完之后，将最终的事件发送给`me-event-display`服务进行事件显示。
具体操作步骤如下：
### 创建 Knative Service
```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: me-even-odd-switcher
spec:
  template:
    spec:
      containers:
      - image: villardl/switcher-nodejs:0.1
        env:
        - name: EXPRESSION
          value: Math.round(Date.parse(event.time) / 60000) % 2
        - name: CASES
          value: '[0, 1]'
---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: even-transformer
spec:
  template:
    spec:
      containers:
      - image: villardl/transformer-nodejs:0.1
        env:
        - name: TRANSFORMER
          value: |
            ({"message": "we are even!"})

---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: odd-transformer
spec:
  template:
    spec:
      containers:
      - image: villardl/transformer-nodejs:0.1
        env:
        - name: TRANSFORMER
          value: |
            ({"message": "this is odd!"})
.
```

### 创建 Parallel
```yaml
apiVersion: messaging.knative.dev/v1alpha1
kind: Parallel
metadata:
  name: me-odd-even-parallel
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
  cases:
    - filter:
        uri: "http://me-even-odd-switcher.default.svc.cluster.local/0"
      subscriber:
        ref:
          apiVersion: serving.knative.dev/v1alpha1
          kind: Service
          name: me-even-transformer
    - filter:
        uri: "http://me-even-odd-switcher.default.svc.cluster.local/1"
      subscriber:
        ref:
          apiVersion: serving.knative.dev/v1alpha1
          kind: Service
          name: me-odd-transformer
  reply:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: me-event-display
```

### 创建 CronJobSource 数据源

```yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: CronJobSource
metadata:
  name: me-cronjob-source
spec:
  schedule: "*/1 * * * *"
  data: '{"message": "Even or odd?"}'
  sink:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: Parallel
    name: me-odd-even-parallel
```
### 查看结果
运行之后可以看到类似如下结果：
```bash
kubectl logs -l serving.knative.dev/service=me-event-display --tail=30 -c user-container

️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 0.3
  type: dev.knative.cronjob.event
  source: /apis/v1/namespaces/default/cronjobsources/me-cronjob-source
  id: 48eea348-8cfd-4aba-9ead-cb024ce16a48
  time: 2019-07-31T20:56:00.000477587Z
  datacontenttype: application/json; charset=utf-8
Extensions,
  knativehistory: me-odd-even-parallel-kn-parallel-kn-channel.default.svc.cluster.local, me-odd-even-parallel-kn-parallel-0-kn-channel.default.svc.cluster.local
Data,
  {
    "message": "we are even!"
  }
️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 0.3
  type: dev.knative.cronjob.event
  source: /apis/v1/namespaces/default/cronjobsources/me-cronjob-source
  id: 42717dcf-b194-4b36-a094-3ea20e565ad5
  time: 2019-07-31T20:57:00.000312243Z
  datacontenttype: application/json; charset=utf-8
Extensions,
  knativehistory: me-odd-even-parallel-kn-parallel-1-kn-channel.default.svc.cluster.local, me-odd-even-parallel-kn-parallel-kn-channel.default.svc.cluster.local
Data,
  {
    "message": "this is odd!"
  }
```

## 结论
通过上面的介绍，相信大家对 Parallel 如何进行事件条件处理有了更多的了解，对于并行处理事件的场景下，不妨试试 Parallel。
