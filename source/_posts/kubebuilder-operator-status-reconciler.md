---
title: Kubebuilder Operator 开发
date: 2024-09-18 21:00:00
tags:
  - kubernetes
  - kubebuilder
  - operator
  - controller-runtime
categories:
  - Cloud Native
---

写 Operator 时，真正难的往往不是「把 Deployment 创出来」，而是把状态闭环、删除收尾、以及「子资源变了也要驱动 reconcile」这几件事做对。CRD 官方只允许两类 subresource：`status` 和 `scale`。下面按这条主线，把 Status、Event、Finalizer，以及 controller-runtime 里常用的扩展点串起来。

## 1. 先把 Status 变成真正的子资源

Kubebuilder 脚手架会生成带 `Spec` / `Status` 字段的类型，但这只是 Go 结构体上的两个成员，**不等于** API Server 已经暴露了 status subresource：

```go
type SampleSpec struct {
	Foo string `json:"foo,omitempty"`
}

type SampleStatus struct {
	// 观测态字段放这里
}

// +kubebuilder:object:root=true
type Sample struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   SampleSpec   `json:"spec,omitempty"`
	Status SampleStatus `json:"status,omitempty"`
}
```

此时在 controller 里调用 `r.Status().Update(...)`，很容易直接 panic，错误类似：

```text
the server could not find the requested resource
```

原因很直接：Pod 那套「改 status 走 `/status`」的路径，对你的 CR 还没打开。

要和内建资源对齐，约定也很清晰：

- 用户只写 `spec`（期望态）
- controller 负责维护 `status`（观测态）
- reconcile 对比两者，决定下一步动作

在类型上补注解即可：

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
type Sample struct {
	// ...
}
```

重新 generate / 安装 CRD 后，`Status().Update()` 才有对应的 API 路径。声明式控制循环里，Status 通常是决策依据，而不只是「给 kubectl 好看」的附属字段。

## 2. 用 Event 补一层可运维信号

Status 适合表达结构化状态；排查现场问题时，Event 往往更直观。`kubectl describe` 能直接把近期事件摊开，相当于把关键路径上的重要日志挂到对象上。

在 Reconciler 里注入 `EventRecorder`：

```go
type SampleReconciler struct {
	client.Client
	Log      logr.Logger
	Scheme   *runtime.Scheme
	Recorder record.EventRecorder
}

func (r *SampleReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	sample := &samplev1.Sample{}
	if err := r.Get(ctx, req.NamespacedName, sample); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// ...业务处理...

	r.Recorder.Eventf(sample, corev1.EventTypeWarning, "ReconcileFailed", "failed to sync: %v", err)
	return ctrl.Result{}, err
}
```

级别就两类：`Normal` 和 `Warning`。别把 Event 当完整审计日志用，关键里程碑、失败原因记清楚就够。

## 3. Finalizer：删除前把尾巴收干净

对象的 `metadata.finalizers` 是字符串列表。只要列表非空，API Server 不会真正清掉对象；它会先打上 `deletionTimestamp`，后续更新会持续进入 reconcile，直到你把自家 finalizer 摘掉。

典型模式：

```go
const myFinalizer = "sample.finalizers.io"

func (r *SampleReconciler) ensureFinalizer(ctx context.Context, sample *samplev1.Sample) error {
	if sample.DeletionTimestamp.IsZero() {
		// 正常存活：确保 finalizer 已挂上
		if !controllerutil.ContainsFinalizer(sample, myFinalizer) {
			controllerutil.AddFinalizer(sample, myFinalizer)
			return r.Update(ctx, sample)
		}
		return nil
	}

	// 正在删除：先清理依赖，再摘 finalizer
	if controllerutil.ContainsFinalizer(sample, myFinalizer) {
		// 例如删掉 CR 创建的 Deployment / PVC / 外部资源
		if err := r.cleanupOwnedResources(ctx, sample); err != nil {
			return err
		}
		controllerutil.RemoveFinalizer(sample, myFinalizer)
		return r.Update(ctx, sample)
	}
	return nil
}
```

没有 finalizer 时，「先删子资源再放行 CR」这件事没法可靠完成；有了它，删除路径就变成普通的 reconcile 分支。

## 4. 事件怎么进队列：EventHandler 与 Predicate

Informer 已经由 controller-runtime 管好了。你真正要关心的是：监听到 Create / Update / Delete 之后，**哪些事件入队**、**入队前能不能插手**。

两个扩展点：

| 扩展点 | 职责 |
| --- | --- |
| `Predicate` | 过滤：这个事件要不要继续处理 |
| `EventHandler` | 映射：事件如何变成 `reconcile.Request` 入队 |

`EventHandler` 接口大致是：

```go
type EventHandler interface {
	Create(event.CreateEvent, workqueue.RateLimitingInterface)
	Update(event.UpdateEvent, workqueue.RateLimitingInterface)
	Delete(event.DeleteEvent, workqueue.RateLimitingInterface)
	Generic(event.GenericEvent, workqueue.RateLimitingInterface)
}
```

`Predicate` 则对四类事件分别返回 bool。源码路径上（以较早版本为例），`OnAdd` 会先跑一遍 Predicates，通过后再调 handler：

```go
func (e EventHandler) OnAdd(obj interface{}) {
	c := event.CreateEvent{ /* ... */ }
	for _, p := range e.Predicates {
		if !p.Create(c) {
			return
		}
	}
	e.EventHandler.Create(c, e.Queue)
}
```

队列里最终只有：

```go
type NamespacedName struct {
	Namespace string
	Name      string
}
```

也就是说：**队列不携带 GVK**。多个来源的事件若都要驱动同一个 CR，必须在 handler 里自己把「子对象事件」翻译成「父 CR 的 name/namespace」。

Controller 启动后会起多个 worker，循环 `Queue.Get()` → `reconcileHandler`。处理完要 `Done`；只有确定不必重试时才 `Forget`，瞬态错误通常留给 workqueue 退避重入。

## 5. 同时盯住 CR 和它创建的资源

常见链路是 `CR → Deployment → Pod`。只 watch CR 不够：Deployment 副本没起来、Pod 挂了，CR 自己可能根本没变更，reconcile 就不会跑。

默认脚手架只绑了主资源：

```go
func (r *SampleReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&samplev1.Sample{}).
		Complete(r)
}
```

builder 提供三层语义（底层都落在 `Watch`）：

- `For`：主资源，事件直接映射到自身
- `Owns`：从资源，且依赖 OwnerReference 反查 owner
- `Watches`：完全自定义 source + handler

粗暴监听所有 Deployment：

```go
Watches(
	&source.Kind{Type: &appsv1.Deployment{}},
	&handler.EnqueueRequestForObject{},
)
```

这会把集群里无关 Deployment 也灌进队列。更常见的是 `Owns`：

```go
return ctrl.NewControllerManagedBy(mgr).
	For(&samplev1.Sample{}).
	Owns(&appsv1.Deployment{}).
	Complete(r)
```

创建子对象时补上归属：

```go
_ = controllerutil.SetControllerReference(sample, deploy, r.Scheme)
```

这样只有挂在该 CR 下的 Deployment 变化，才会反过来 enqueue 对应的 Sample。

### 自定义 handler：把子资源变化映射回 CR

Reconcile 的业务主体仍应是 CR。子资源更新时，CR 未必有 Update 事件，所以 handler 需要分支：

- 对象本身是 Sample → 入队该 Sample
- 对象是 Deployment → 读 `OwnerReferences`，入队 owner 对应的 Sample

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

`Owns()` 内部做的，本质上就是这类「按 owner 回指」的 handler；自己实现时记得把 Create / Update / Delete 都覆盖完整。

## 6. 只关心某类字段变更：自定义 Predicate

不想被无关 Update 打爆队列时，用 Predicate 收窄。例如「只有 Sample 的 labels 变了才 reconcile」：

```go
type ResourceLabelChangedPredicate struct {
	predicate.Funcs
}

func (p ResourceLabelChangedPredicate) Update(e event.UpdateEvent) bool {
	oldObj, ok1 := e.ObjectOld.(*samplev1.Sample)
	newObj, ok2 := e.ObjectNew.(*samplev1.Sample)
	if !(ok1 && ok2) {
		return false
	}
	return !mapsEqual(oldObj.GetLabels(), newObj.GetLabels())
}
```

一旦同时 watch 了多种资源，Predicate 也会收到 Deployment 等事件，**先做类型断言再比字段**，否则会误伤或误放行。

## 7. CRD 多版本与 Conversion Webhook

API 演进几乎必然带来多版本并存。Kubebuilder 可以通过 conversion webhook，让同一份存储数据以不同版本对外呈现。例如创建了 v2 对象后：

```bash
kubectl apply -f config/samples/batch_v2_cronjob.yaml

kubectl get cronjobs.v2.batch.tutorial.kubebuilder.io -o yaml
kubectl get cronjobs.v1.batch.tutorial.kubebuilder.io -o yaml
```

两次 get 字段布局可以不同，中间的差异由 conversion webhook 负责双向转换。设计多版本时，尽早定 storage version，并保证字段迁移可逆、可测。

## 8. Webhook 里读集群：补一个可用的 Client

Admission 逻辑经常要查别的对象。比如自定义 `PodBox`，约束「每个 PodBox 下最多一个 Pod」，`ValidateCreate` 就得按归属 list/count。

Kubebuilder 2.x 对 webhook 封装偏薄，一个实用做法是在 API 包里挂全局 client，启动时注入：

```go
package v1

import (
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
)

var (
	globalClient client.Client
	globalReader client.Reader
)

func InitClient(mgr ctrl.Manager) {
	globalClient = mgr.GetClient()
	globalReader = mgr.GetAPIReader()
}
```

在 `main.go` 里，各 `SetupWithManager` 之前调用 `InitClient(mgr)`，webhook 校验里就可以直接用 `globalReader` / `globalClient`。需要强一致读时优先 `GetAPIReader()`，避免 cache 尚未同步带来的误判。

## 9. 给内建对象挂自定义 Webhook

除了自家 CR，有时还要对原生 Pod 做注入（写 annotation），或拦截更新/删除。社区有 builtins 示例可参考：

[controller-runtime validatingwebhook example](https://github.com/kubernetes-sigs/controller-runtime/blob/master/examples/builtins/validatingwebhook.go)

落地时有个坑：接入后跑 `make generate` 可能报：

```text
invalid field type interface{sigs.k8s.io/controller-runtime/pkg/client.Reader; ...}
```

实践里，跳过 generate、继续 `make docker-build` 一类流程往往仍能过。若必须 generate，需要把 webhook 结构体字段类型整理成 codegen 能识别的形式，而不是直接塞完整的 client 接口组合。

---

小结一下优先级：

1. 打开 status subresource，让观测态更新走对路径
2. Finalizer 管删除闭环，Event 管现场可读性
3. 用 `Owns` / 自定义 handler 把子资源变化折回 CR
4. 用 Predicate 砍掉噪声事件
5. 多版本靠 conversion；webhook 里查集群要显式注入 client

把这几块接上，Operator 才更像一个可演进、可运维的控制面，而不是「创建资源的脚本」。
