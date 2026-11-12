---
title: Kubebuilder 开发笔记：Status、Reconciler 与 Webhook
date: 2024-09-18 21:00:00
tags:
  - kubernetes
  - kubebuilder
  - operator
  - controller-runtime
categories:
  - Cloud Native
---

整理一下用 Kubebuilder 写 Operator 时常碰到的几块内容：CRD 的 Status 子资源、Event 与 Finalizer、controller-runtime 的入队过滤和多资源监听，以及多版本和 Webhook 里的一些处理办法。

## Status 子资源

K8s 里每种自定义资源最多只能带两种子资源：`status` 和 `scale`。

Kubebuilder 初始化完 API 后，结构体里已经把 `Spec` 和 `Status` 都生成好了，但这时候 `Status` 还只是普通字段，并不是真正的 subresource：

```go
type SampleSpec struct {
	Foo string `json:"foo,omitempty"`
}

type SampleStatus struct {
}

// +kubebuilder:object:root=true
type Sample struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   SampleSpec   `json:"spec,omitempty"`
	Status SampleStatus `json:"status,omitempty"`
}
```

如果直接在这种状态下调 `r.Status().Update()`，运行期会报错：`the server could not find the requested resource`。因为默认情况下它还不像 `Pod.Status` 那样拥有独立的 `/status` 接口。

按照 K8s 的设计习惯，用户只改 `spec`，控制器负责回写 `status`。Reconcile 里也是靠读取当前 `status` 跟 `spec` 对比，来决定下一步动作。想按这套方式用，需要在 `Sample` 上加一行注解：

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
type Sample struct {
	// ...
}
```

加完重新执行 `make manifests` 更新 CRD，再去调 `Status().Update()` 就正常了。

## 记录 Event

`Status` 用来驱动控制逻辑，平时排查问题则经常要看 Event，作用跟在关键业务节点打日志差不多，跑起来之后用 `kubectl describe` 就能直接看到。

用法很简单，在 `SampleReconciler` 里加个 `record.EventRecorder` 字段：

```go
type SampleReconciler struct {
	client.Client
	Log      logr.Logger
	Scheme   *runtime.Scheme
	Recorder record.EventRecorder
}

func (r *SampleReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	sample := &samplev1.Sample{}
	_ = r.Get(ctx, req.NamespacedName, sample)

	// 业务逻辑...

	r.Recorder.Eventf(sample, corev1.EventTypeWarning, "Error", "some error")
	return ctrl.Result{}, nil
}
```

K8s 提供的 Event 等级就两种：`Normal` 和 `Warning`。

## Finalizer

Finalizer 对应对象上的 `metadata.finalizers` 字段，是个字符串数组。

删除一个带 finalizer 的资源时，K8s 不会马上把它删掉，而是先设置 `DeletionTimestamp`，这会触发一次 Update 事件。Controller 监听到这个字段非空，就可以先做一些清理动作（比如把 CR 创建出来的其他资源删掉），处理完了再把 finalizer 从列表里去掉，对象才会真正消失。

代码里一般单独抽个函数在 Reconcile 开头跑：

```go
func (r *SampleReconciler) finalize(ctx context.Context, sample *samplev1.Sample) error {
	myFinalizerName := "sample.finalizers.io"

	if sample.ObjectMeta.DeletionTimestamp.IsZero() {
		// 没在删除，检查有没有挂上 finalizer，没有就补上
		if !containsString(sample.ObjectMeta.Finalizers, myFinalizerName) {
			sample.ObjectMeta.Finalizers = append(sample.ObjectMeta.Finalizers, myFinalizerName)
			return r.Update(ctx, sample)
		}
		return nil
	}

	// DeletionTimestamp 不为空，走预删除逻辑
	if containsString(sample.ObjectMeta.Finalizers, myFinalizerName) {
		// 执行清理动作...

		sample.ObjectMeta.Finalizers = removeString(sample.ObjectMeta.Finalizers, myFinalizerName)
		return r.Update(ctx, sample)
	}
	return nil
}
```

## EventHandler 与 Predicate

如果要定制事件监听逻辑，得先搞清楚 controller-runtime 在事件进 WorkQueue 之前做了什么。

Informer 本身框架已经封装好了。当监听到对象的增删改事件时，进队列前主要经过 `Predicate` 和 `EventHandler` 两个接口。

`Predicate` 负责过滤，决定某次事件要不要处理：

```go
type Predicate interface {
	Create(event.CreateEvent) bool
	Delete(event.DeleteEvent) bool
	Update(event.UpdateEvent) bool
	Generic(event.GenericEvent) bool
}
```

`EventHandler` 则是在事件入队前做处理，决定往队列里塞什么内容：

```go
type EventHandler interface {
	Create(event.CreateEvent, workqueue.RateLimitingInterface)
	Update(event.UpdateEvent, workqueue.RateLimitingInterface)
	Delete(event.DeleteEvent, workqueue.RateLimitingInterface)
	Generic(event.GenericEvent, workqueue.RateLimitingInterface)
}
```

翻一下 `eventsource.go` 就能看到两者的串联关系。以 `OnAdd` 为例，先跑一遍所有 `Predicates`，只要有一个返回 `false` 就直接返回，全过了才调 `EventHandler.Create` 入队（`OnUpdate`、`OnDelete` 也一样）：

```go
type EventHandler struct {
	EventHandler handler.EventHandler
	Queue        workqueue.RateLimitingInterface
	Predicates   []predicate.Predicate
}

func (e EventHandler) OnAdd(obj interface{}) {
	c := event.CreateEvent{}
	// ...
	for _, p := range e.Predicates {
		if !p.Create(c) {
			return
		}
	}
	e.EventHandler.Create(c, e.Queue)
}
```

这里要注意，最终进 WorkQueue 的结构只有 `Namespace` 和 `Name`，不包含资源类型：

```go
type NamespacedName struct {
	Namespace string
	Name      string
}
```

消费端在 `controller.Start()` 里，会根据 `MaxConcurrentReconciles` 启动多个 worker 协程，循环调 `processNextWorkItem` 从队列拿任务交给 `reconcileHandler`：

```go
func (c *Controller) processNextWorkItem() bool {
	obj, shutdown := c.Queue.Get()
	if shutdown {
		return false
	}
	defer c.Queue.Done(obj)

	return c.reconcileHandler(obj)
}
```

每次处理完要调 `Done`；如果处理成功、不需要重试，内部还会调 `Forget`，否则遇上临时错误会按退避时间重新放回队列。

## 监听多个资源与字段过滤

很多时候业务链路是 `CR -> Deployment -> Pod`，光盯住 CR 不够，底下的 Deployment 或 Pod 变了也得感知到。

默认生成的 `SetupWithManager` 只写了 `For`：

```go
func (r *SampleReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&samplev1.Sample{}).
		Complete(r)
}
```

看 `pkg/builder/controller.go` 的源码可以发现，`For()` 和 `Owns()` 底层都是调 `Watches()`，只是各自配了不同的 `EventHandler`。

如果想监听 Deployment，可以直接写 `Watches`：

```go
func (r *SampleReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&samplev1.Sample{}).
		Watches(&source.Kind{Type: &appsv1.Deployment{}}, &handler.EnqueueRequestForObject{}).
		Complete(r)
}
```

不过这样会把集群里所有 Deployment 的事件都收进来。如果只关心当前 CR 创建出来的 Deployment，用 `Owns` 就行：

```go
func (r *SampleReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&samplev1.Sample{}).
		Owns(&appsv1.Deployment{}).
		Complete(r)
}
```

同时在创建 Deployment 的时候把从属关系写进 `OwnerReferences`：

```go
_ = controllerutil.SetControllerReference(sample, &deployment, r.Scheme)
```

这样就只会处理带有对应 Ownership 的 Deployment 事件了。

### 多资源下的自定义 EventHandler

前面提到队列里只有 `NamespacedName`，而 Reconciler 里的主逻辑是围绕 CR 写的。如果 Deployment 变了，CR 本身并不会产生更新事件，所以自定义 `EventHandler` 时需要判断一下事件对象的类型：是 CR 就直接把它的名字放进队列；是 Deployment 就顺着 `OwnerReferences` 找到对应的 CR 名字再入队，从而触发那条 CR 的 Reconcile：

```go
func (e *EnqueueRequestForDP) Create(evt event.CreateEvent, q workqueue.RateLimitingInterface) {
	if evt.Meta == nil {
		return
	}
	if _, ok := evt.Object.(*samplev1.Sample); ok {
		q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
			Name:      evt.Meta.GetName(),
			Namespace: evt.Meta.GetNamespace(),
		}})
		return
	}

	deploy, ok := evt.Object.(*appsv1.Deployment)
	if !ok {
		return
	}
	for _, owner := range deploy.OwnerReferences {
		if owner.Kind == "Sample" {
			q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
				Name:      owner.Name,
				Namespace: evt.Meta.GetNamespace(),
			}})
		}
	}
}
```

`Owns()` 内部其实也是靠类似的 handler 实现的。

### 只监听特定字段变化

如果只想关注某几个字段的改动，比如 CR 的 label 变了才处理，写个自定义 `Predicate` 实现 `Update` 方法即可：

```go
func (rl *ResourceLabelChangedPredicate) Update(e event.UpdateEvent) bool {
	_, ok1 := e.ObjectOld.(*samplev1.Sample)
	_, ok2 := e.ObjectNew.(*samplev1.Sample)
	if ok1 && ok2 {
		if !compareMaps(e.MetaOld.GetLabels(), e.MetaNew.GetLabels()) {
			return true
		}
	}
	return false
}
```

注意一旦 controller 同时监听了多个资源，Deployment 的更新也会进到这个函数里，所以开头要先做类型判断。

## 多版本与 Webhook

### 多版本转换

CRD 往后演进免不了出现多个版本共存。Kubebuilder 支持用 conversion webhook 做不同版本间的读写转换。比如提交了一个 v2 的 CronJob：

```bash
kubectl apply -f config/samples/batch_v2_cronjob.yaml
```

之后不管指定 v1 还是 v2 去查，都能正常读出来：

```bash
kubectl get cronjobs.v2.batch.tutorial.kubebuilder.io -o yaml
kubectl get cronjobs.v1.batch.tutorial.kubebuilder.io -o yaml
```

两边拿到的 YAML 字段会有差异，中间的数据转换就是由 conversion webhook 完成的。

### 在 Webhook 里访问集群资源

有时候做准入校验光看当前对象不够，还得查集群里的其他资源。比如定义了一个 `PodBox`，要求每个 `PodBox` 下面最多只能有一个 Pod，那在 `ValidateCreate` 里就得把关联的 Pod 列出来数一下。

Kubebuilder 2.x 把 webhook 封装得很死，方法签名里没给 client。一个绕开的办法是在 types / webhook 目录下建个文件，挂一组包级 client：

```go
package v1

import (
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
)

var globalClient client.Client
var globalReader client.Reader

func InitClient(mgr ctrl.Manager) {
	globalClient = mgr.GetClient()
	globalReader = mgr.GetAPIReader()
}
```

在 `main.go` 里、各个 `SetupWithManager` 调用之前先执行一次 `InitClient(mgr)`，之后 `ValidateCreate` 里就能直接拿来用了。

### 给原生对象加自定义 Webhook

除了拦截自己的 CRD，有时也需要给原生 Pod 注入 annotation，或者校验原生资源的更新、删除。社区给过一个针对内置对象的写法：[validatingwebhook.go](https://github.com/kubernetes-sigs/controller-runtime/blob/master/examples/builtins/validatingwebhook.go)。

不过按这个例子接进项目后，执行 `make generate` 会报错：

```text
invalid field type interface{sigs.k8s.io/controller-runtime/pkg/client.Reader; sigs.k8s.io/controller-runtime/pkg/client.StatusClient; sigs.k8s.io/controller-runtime/pkg/client.Writer}
```

试了一下，只要不跑 `make generate`，像 `make docker-build` 等其他步骤都可以正常走通。
