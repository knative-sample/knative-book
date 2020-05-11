---
title: "关于 Broker/Trigger 事件模型"
date: 2020-5-10T15:26:15Z
draft: false
weight: 20
description: "calling built-in Shortcodes into your content files."
---

## Broker 和 Trigger
从 v0.5 开始，Knative Eventing 定义 Broker 和 Trigger 对象，从而能方便的对事件进行过滤（亦如通过 ingress 和 ingress controller 对网络流量的过滤一样）。
* Broker 提供一个事件集，可以通过属性选择该事件集。它负责接收事件并将其转发给由一个或多个匹配 Trigger 定义的订阅者。
* Trigger 描述基于事件属性的过滤器。同时可以根据需要创建多个 Trigger。
如图：
![broker-trigger.png](https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/6bfef3fca5a279e93230b498fdd984bf.png)

## 当前实现方式
### Namespace
通过`Namespace Reconciler` (代码：eventing/pkg/reconciler/v1alpha1/namespace/namespace.go)创建 broker。`Namespace Reconciler` 会查询所有带`knative-eventing-injection: enabled` 标签的 namespace。如果存在这样标签的 namespace, 那么`Namespace Reconciler`将会进行如下处理操作：
1. 创建 Broker 过滤器的 ServiceAccount：eventing-broker-filter
2. 通过 RoleBinding 确保 ServiceAccount 的 RBAC 权限
3. 创建名称为 default 的 Broker

```go
// newBroker creates a placeholder default Broker object for Namespace 'ns'.
func newBroker(ns *corev1.Namespace) *v1alpha1.Broker {
	return &v1alpha1.Broker{
		ObjectMeta: metav1.ObjectMeta{
			Namespace: ns.Name,
			Name:      defaultBroker,
			Labels:    injectedLabels(),
		},
	}
}
```
### Broker (事件代理)
通过` Broker Reconciler`进行处理 broker，对于每一个 broker, 会进行一下处理操作：
1. 创建 'trigger'`Channel`。所有在 Broker 中的 event 事件都会发送到这个`Channel`， 所有的 Trigger 会订阅这个`Channel`。
2. 创建'filter'`Deployment`。这个 Deployment 会运行`cmd/broker/filter`。其目的是处理与此 Broker 相关的所有 Trigger 的数据平面。说白了其实就做了两件事情，从 Channel 中接收事件，然后转发给事件的订阅者。
3. 创建'filter' Kubernetes Service。通过该 Service 提供'filter' `Deployment`的服务访问。
4. 创建'ingress' Deployment。这个 Deployment 会运行 cmd/broker/ingress。其目的是检查进入 Broker的所有事件
5. 创建'ingress' Kubernetes Service。通过该 Service提供'Ingress' `Deployment`的服务访问。
6. 创建'ingress' Channel。这是一个 Trigger 应答的 Channel。目的是将 Trigger 中返回的事件通过 Ingress Deployment 回写到 Broker。理想情况下，其实不需要这个，可以直接将 Trigger 的响应发送给 k8s Service。但是作为订阅的场景，只允许我们向 Channel 发送响应信息，所以我们需要这个作为中介。
7. 创建'ingress' Subscription。它通过'ingress' Channel来订阅'ingress' Service
代码如下：
```go
func (r *reconciler) reconcile(ctx context.Context, b *v1alpha1.Broker) (reconcile.Result, error) {
	// 1. Trigger Channel is created for all events. Triggers will Subscribe to this Channel.
	// 2. Filter Deployment.
	// 3. Ingress Deployment.
	// 4. K8s Services that point at the Deployments.
	// 5. Ingress Channel is created to get events from Triggers back into this Broker via the
	//    Ingress Deployment.
	//   - Ideally this wouldn't exist and we would point the Trigger's reply directly to the K8s
	//     Service. However, Subscriptions only allow us to send replies to Channels, so we need
	//     this as an intermediary.
	// 6. Subscription from the Ingress Channel to the Ingress Service.

	if b.DeletionTimestamp != nil {
		// Everything is cleaned up by the garbage collector.
		return nil
	}

	if b.Spec.ChannelTemplate == nil {
		r.Logger.Error("Broker.Spec.ChannelTemplate is nil",
			zap.String("namespace", b.Namespace), zap.String("name", b.Name))
		return nil
	}

	gvr, _ := meta.UnsafeGuessKindToResource(b.Spec.ChannelTemplate.GetObjectKind().GroupVersionKind())
	channelResourceInterface := r.DynamicClientSet.Resource(gvr).Namespace(b.Namespace)
	if channelResourceInterface == nil {
		return fmt.Errorf("unable to create dynamic client for: %+v", b.Spec.ChannelTemplate)
	}

	track := r.channelableTracker.TrackInNamespace(b)

	triggerChannelName := resources.BrokerChannelName(b.Name, "trigger")
	triggerChannelObjRef := corev1.ObjectReference{
		Kind:       b.Spec.ChannelTemplate.Kind,
		APIVersion: b.Spec.ChannelTemplate.APIVersion,
		Name:       triggerChannelName,
		Namespace:  b.Namespace,
	}
	// Start tracking the trigger channel.
	if err := track(triggerChannelObjRef); err != nil {
		return fmt.Errorf("unable to track changes to the trigger Channel: %v", err)
	}

	logging.FromContext(ctx).Info("Reconciling the trigger channel")
	triggerChan, err := r.reconcileTriggerChannel(ctx, channelResourceInterface, triggerChannelObjRef, b)
	if err != nil {
		logging.FromContext(ctx).Error("Problem reconciling the trigger channel", zap.Error(err))
		b.Status.MarkTriggerChannelFailed("ChannelFailure", "%v", err)
		return err
	}
   ......

	filterDeployment, err := r.reconcileFilterDeployment(ctx, b)
	if err != nil {
		logging.FromContext(ctx).Error("Problem reconciling filter Deployment", zap.Error(err))
		b.Status.MarkFilterFailed("DeploymentFailure", "%v", err)
		return err
	}
	_, err = r.reconcileFilterService(ctx, b)
	if err != nil {
		logging.FromContext(ctx).Error("Problem reconciling filter Service", zap.Error(err))
		b.Status.MarkFilterFailed("ServiceFailure", "%v", err)
		return err
	}
	b.Status.PropagateFilterDeploymentAvailability(filterDeployment)

	ingressDeployment, err := r.reconcileIngressDeployment(ctx, b, triggerChan)
	if err != nil {
		logging.FromContext(ctx).Error("Problem reconciling ingress Deployment", zap.Error(err))
		b.Status.MarkIngressFailed("DeploymentFailure", "%v", err)
		return err
	}

	svc, err := r.reconcileIngressService(ctx, b)
	if err != nil {
		logging.FromContext(ctx).Error("Problem reconciling ingress Service", zap.Error(err))
		b.Status.MarkIngressFailed("ServiceFailure", "%v", err)
		return err
	}
	b.Status.PropagateIngressDeploymentAvailability(ingressDeployment)
	b.Status.SetAddress(&apis.URL{
		Scheme: "http",
		Host:   names.ServiceHostName(svc.Name, svc.Namespace),
	})

	ingressChannelName := resources.BrokerChannelName(b.Name, "ingress")
	ingressChannelObjRef := corev1.ObjectReference{
		Kind:       b.Spec.ChannelTemplate.Kind,
		APIVersion: b.Spec.ChannelTemplate.APIVersion,
		Name:       ingressChannelName,
		Namespace:  b.Namespace,
	}

	// Start tracking the ingress channel.
	if err = track(ingressChannelObjRef); err != nil {
		return fmt.Errorf("unable to track changes to the ingress Channel: %v", err)
	}

	ingressChan, err := r.reconcileIngressChannel(ctx, channelResourceInterface, ingressChannelObjRef, b)
	if err != nil {
		logging.FromContext(ctx).Error("Problem reconciling the ingress channel", zap.Error(err))
		b.Status.MarkIngressChannelFailed("ChannelFailure", "%v", err)
		return err
	}
	b.Status.IngressChannel = &ingressChannelObjRef
	b.Status.PropagateIngressChannelReadiness(&ingressChan.Status)

	ingressSub, err := r.reconcileIngressSubscription(ctx, b, ingressChan, svc)
	if err != nil {
		logging.FromContext(ctx).Error("Problem reconciling the ingress subscription", zap.Error(err))
		b.Status.MarkIngressSubscriptionFailed("SubscriptionFailure", "%v", err)
		return err
	}
	b.Status.PropagateIngressSubscriptionReadiness(&ingressSub.Status)

	return nil
}
```
Broker 示例：

```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: Broker
metadata:
  name: default
spec:
  channelTemplateSpec:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
```

### Trigger (触发器)
通过` Trigger Reconciler`进行处理 trigger，对于每一个 trigger, 会进行一下处理操作：
1. 验证 Broker 是否存在
2. 获取对应 Broker 的 Trigger Channel、 Ingress Channel 以及 Filter Service
3. 确定订阅者的 URI
4. 创建一个从 Broker 特定的 Channel 到这个 Trigger 的 kubernetes Service 的订阅。reply 被发送到 
 Broker 的 `ingress` Channel。
 5. 检查是否包含 `knative.dev/dependency` 的注释。
代码如下：
```go
func (r *reconciler) reconcile(ctx context.Context, t *v1alpha1.Trigger) error {
    ......
    // 1. Verify the Broker exists.
	// 2. Get the Broker's:
	//   - Trigger Channel
	//   - Ingress Channel
	//   - Filter Service
	// 3. Find the Subscriber's URI.
	// 4. Creates a Subscription from the Broker's Trigger Channel to this Trigger via the Broker's
	//    Filter Service with a specific path, and reply set to the Broker's Ingress Channel.
	// 5. Find whether there is annotation with key "knative.dev/dependency".
	// If not, mark Dependency to be succeeded, else figure out whether the dependency is ready and mark Dependency correspondingly

	if t.DeletionTimestamp != nil {
		// Everything is cleaned up by the garbage collector.
		return nil
	}
	// Tell tracker to reconcile this Trigger whenever the Broker changes.
	brokerObjRef := corev1.ObjectReference{
		Kind:       brokerGVK.Kind,
		APIVersion: brokerGVK.GroupVersion().String(),
		Name:       t.Spec.Broker,
		Namespace:  t.Namespace,
	}

	if err := r.tracker.Track(brokerObjRef, t); err != nil {
		logging.FromContext(ctx).Error("Unable to track changes to Broker", zap.Error(err))
		return err
	}

	b, err := r.brokerLister.Brokers(t.Namespace).Get(t.Spec.Broker)
	if err != nil {
		logging.FromContext(ctx).Error("Unable to get the Broker", zap.Error(err))
		if apierrs.IsNotFound(err) {
			t.Status.MarkBrokerFailed("DoesNotExist", "Broker does not exist")
			_, needDefaultBroker := t.GetAnnotations()[v1alpha1.InjectionAnnotation]
			if t.Spec.Broker == "default" && needDefaultBroker {
				if e := r.labelNamespace(ctx, t); e != nil {
					logging.FromContext(ctx).Error("Unable to label the namespace", zap.Error(e))
				}
			}
		} else {
			t.Status.MarkBrokerFailed("BrokerGetFailed", "Failed to get broker")
		}
		return err
	}
	t.Status.PropagateBrokerStatus(&b.Status)

	brokerTrigger := b.Status.TriggerChannel
	if brokerTrigger == nil {
		logging.FromContext(ctx).Error("Broker TriggerChannel not populated")
		r.Recorder.Eventf(t, corev1.EventTypeWarning, triggerChannelFailed, "Broker's Trigger channel not found")
		return errors.New("failed to find Broker's Trigger channel")
	}

	brokerIngress := b.Status.IngressChannel
	if brokerIngress == nil {
		logging.FromContext(ctx).Error("Broker IngressChannel not populated")
		r.Recorder.Eventf(t, corev1.EventTypeWarning, ingressChannelFailed, "Broker's Ingress channel not found")
		return errors.New("failed to find Broker's Ingress channel")
	}

	// Get Broker filter service.
	filterSvc, err := r.getBrokerFilterService(ctx, b)
	......

	subscriberURI, err := r.uriResolver.URIFromDestination(*t.Spec.Subscriber, t)
	if err != nil {
		logging.FromContext(ctx).Error("Unable to get the Subscriber's URI", zap.Error(err))
		return err
	}
	t.Status.SubscriberURI = subscriberURI

	sub, err := r.subscribeToBrokerChannel(ctx, t, brokerTrigger, brokerIngress, filterSvc)
	if err != nil {
		logging.FromContext(ctx).Error("Unable to Subscribe", zap.Error(err))
		t.Status.MarkNotSubscribed("NotSubscribed", "%v", err)
		return err
	}
	t.Status.PropagateSubscriptionStatus(&sub.Status)

	if err := r.checkDependencyAnnotation(ctx, t); err != nil {
		return err
	}

	return nil
}
```
Trigger 示例：

```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: my-service-trigger
spec:
  broker: default
  filter:
    attributes:
      type: dev.knative.foo.bar
      myextension: my-extension-value
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: my-service
```
## 总结
Broker/Trigger 模型出现的意义不仅在于其提供了消息过滤机制，更充分解耦了消息通道的实现，目前除了系统自身支持的基于内存的消息通道 InMemoryChannel 之外，还支持 Kafka、NATS Streaming 等消息服务。
此外结合 CloudEvent 进行事件统一标准传输，无论对于客户端接入事件源，还是消费端提供的消费事件服务，都能极大的提升了应用的跨平台可移植性。


