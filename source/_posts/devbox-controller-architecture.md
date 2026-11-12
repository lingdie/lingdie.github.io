---
title: DevBox 总结（一）：Controller 双层架构与状态机设计
date: 2025-07-14 21:18:00
tags:
  - kubernetes
  - operator
  - controller-runtime
  - devbox
categories:
  - Cloud Native
---

DevBox 做的是跑在 K8s 上的云端开发环境。和普通无状态服务不太一样，开发环境有两个硬需求：一是用户在容器里装的包、改的代码不能因为 Pod 重建就丢；二是实例在不活跃时要能停掉省计算资源，下次拉起还能接着用，甚至能把当前环境打成镜像发版（`DevBoxRelease`）。

如果全靠挂载网络存储（PVC）来做，只有 `/home` 之类特定目录能持久化，`apt install` 写进根文件系统的内容留不住；所以我们底层走的是自定义 snapshotter + 镜像 commit 的路子。但这会直接影响 Controller 的架构设计——因为 K8s 资源是集群全局的，而容器的读写层（RW layer）和 containerd socket 是节点本地的。

这篇主要记录一下 DevBox Controller 在控制面拆分、调度协同、状态流转以及发版 retag 上的做法。

## 为什么要拆成全局 Controller 和节点 DaemonSet

最早如果把所有逻辑塞进一个中心化的 Deployment Controller，很快就会碰壁：Pod 停掉以后，要把它在宿主机上的读写层 commit 成镜像并推送到仓库，就必须连那台机器上的 `/run/containerd/containerd.sock`，清理本地快照和 LVM 逻辑卷也是一样。中心 Controller 没法直接跨节点干这些活。

反过来，如果只跑一个 DaemonSet，让每个节点自己管全套 K8s 资源（Pod、Service、Secret、RBAC），没有选主的情况下多副本同时写同一个 CR 的 Status 又很容易打架。

最后落下来的方案是把控制面拆成两层，共用同一套代码仓库但按角色启动：

1. **全局 `DevboxReconciler`（Deployment，开启 Leader Election）**：只跟 APIServer 打交道，管 Secret、Startup ConfigMap、Service、KubeAccess RBAC 和 Pod 的编排，维护 `Devbox` 的整体状态和 Condition，完全不碰 containerd。
2. **节点 `DevboxNodeReconciler`（`commit-worker` DaemonSet，不选主）**：每台工作节点跑一个实例，挂载宿主机的 containerd socket 和数据目录，通过 Downward API 拿到 `NODE_NAME`。它虽然也 watch `Devbox` CR，但只认领 `currentRecord.Node == r.NodeName` 的对象，专门干节点本地的 commit、push、存储清理以及 RW 层容量采集。

两边不通过 RPC 通信，完全靠 `Devbox.Status` 里的 `contentID` 和 `commitRecords` 做交接。

## 把节点选择交还给 kube-scheduler

早期做本地存储绑定的控制器，很容易写成“Controller 自己先挑一台节点写进 Status，再把 `nodeName` 硬编码进 Pod Spec”。这样写一开始省事，后面接 GPU（比如原生 `nvidia.com/gpu.product` 或 HAMi 调度）、节点亲和性、扩展资源配额时就会非常别扭，等于把调度器干的活在 Controller 里重抄一遍，还经常因为缓存延迟选出根本排不上去的节点。

在 `v1alpha2` 里我们改成了观察式绑定：

- 当 `spec.state == Running` 且当前 `commitRecord.Node` 为空时，全局 Controller 只同步前置依赖（Secret、启动配置、RBAC）并创建不带硬指定节点的 Pod，让 `kube-scheduler` 正常走完过滤和打分。
- Pod 被调度器绑定到具体节点后，`syncAssignedNodeFromPod` 把 `pod.Spec.NodeName` 反写回当前 `commitRecord.Node`，然后立刻 `Requeue` 一轮，再跑完整的同步流水线。
- 后续如果只是 `Paused -> Running` 恢复，因为本地 LV 还在原节点上，Controller 才会根据记录里的节点信息把新 Pod 固定回原节点；如果是 `Stopped` 之后再启动，数据已经进镜像仓库了，就生成新的 `contentID`，重新交给调度器选节点，实现跨节点漂移。

## 四种状态与 contentID 演进

`DevboxSpec.State` 定义了四个状态，区别主要落在“算力释放到哪一步”和“本地存储留不留”：

- `Running`：Pod 正常运行，读写层挂在当前节点的 LVM 瘦供给卷（thin LV）上。
- `Paused`：删掉 Pod 释放 CPU / 内存 / GPU，但**保留节点本地的 LV**（NodePort 等网络资源也保留）。恢复的时候不需要拉取新镜像层，直接把原 LV 重新挂回去，拉起速度最快，代价是不能换节点。
- `Stopped`：删掉 Pod，并且把当前读写层 commit 成镜像推到内部仓库，随后卸载并清理节点本地 LV。下次启动以刚提交的镜像作为新的 `baseImage`，可以调度到任意节点。
- `Shutdown`：在 `Stopped` 的基础上把关联的 Service 等外围资源也释放掉，归档休眠场景用得多。

为了把多次停启串起来，Status 里维护了一个 `contentID`（UUID）和 `commitRecords` 映射表：

```go
// status.contentID 指向当前生效的记录
devbox.Status.CommitRecords[currentContentID] = &devboxv1alpha2.CommitRecord{
	Node:             "",
	BaseImage:        devbox.Spec.Image,
	CommitImage:      r.generateImageName(devbox),
	CommitStatus:     devboxv1alpha2.CommitStatusPending,
	GenerateTime:     metav1.Now(),
	RuntimeClassName: runtimeClassName,
}
```

每次创建 Pod 时，`contentID` 会通过 `devbox.sealos.io/content-id` 注解传给容器，底层 snapshotter 看到这个标识就知道该新建还是复用哪块 LV。

### 状态切换时的跨组件握手

最容易出竞态的地方是 `Running/Paused -> Stopped/Shutdown`。此时如果全局 Controller 直接把 `status.state` 改成 `Stopped`，或者不等 Pod 退出就开始 commit，文件系统可能还有脏页没刷完，甚至容器进程还在写盘。

实际流程是这么卡住顺序的：

1. 用户把 `spec.state` 改成 `Stopped`。
2. 全局 `DevboxReconciler` 在 `runSyncPipeline` 里先把 Pod 删掉；到了 `syncDevboxStateTransition` 这一步，判断出这次转换属于 `devboxStateTransitionNeedsLocalCommit`（目标态是 `Stopped/Shutdown`，当前态是 `Running/Paused` 且已经有归属节点），于是**不推进 `status.state`**，而是打上 `StateTransitionPending=True` 的 Condition（`waiting for node-local commit worker to persist content`），同时也不推进 `ObservedGeneration`。
3. 所属节点上的 `DevboxNodeReconciler` 监听到变更，先调 `devboxPodsGone` 确认这个 DevBox 对应的 Pod 已经彻底消失；如果还没退干净，就 `RequeueAfter: 10s` 继续等。
4. Pod 退干净后，节点 worker 用 `sync.Map` 按 `namespace/name/contentID` 做一层进程内去重，先把 `CommitStatus` 更新成 `Committing`（前端和 API 层看到这个状态会直接拦截新的启停请求，返回 409，避免提交中途又把实例拉起来），然后调 containerd 做 commit + push。
5. 镜像推成功、旧快照和 LV 标记清理完以后，由节点 worker 生成下一轮要用的新 `contentID` 和新 `CommitRecord`（其 `BaseImage` 指向刚刚推上去的 `CommitImage`），最后把 `status.state` 改成目标态。

删除 `Devbox` 时也是类似的协同逻辑：全局 Controller 在 `reconcileDevboxDeletion` 里先把关联的 Pod、Service、ConfigMap、Secret 删掉（摘 Pod finalizer 时还会校验前后的 `Pod.UID` 防止误摘同名重建的 Pod），然后检查 `localStorageCleanupNode`。只要本地存储还没清完，就保留 `Devbox` 上的 finalizer 每 10 秒重入一次；等节点上的 `DevboxNodeReconciler` 确认 Pod 退出并调 `cleanupStorage` 把本地快照和 LV 回收干净后，全局 Controller 才最终摘掉 `devbox.sealos.io/finalizer`。

## Pipeline 组织同步步骤与 KubeAccess

全局 Controller 要同步的子资源不少，如果全平铺在 `Reconcile` 里，打日志、发 Event、更新每种资源的 Condition 会写出一大堆重复模板代码。我们在 `devbox_sync_pipeline.go` 里抽了一层简单的声明式 step 结构：

```go
func (r *DevboxReconciler) syncPipelineSteps(
	devbox *devboxv1alpha2.Devbox,
	recLabels map[string]string,
) []devboxSyncPipelineStep {
	return []devboxSyncPipelineStep{
		r.syncSecretStep(devbox, recLabels),
		r.syncStartupConfigMapStep(devbox, recLabels),
		r.syncNetworkStep(devbox, recLabels),
		r.syncDevboxPhaseStep(devbox, recLabels),
		r.syncKubeAccessStep(devbox, recLabels),
		r.syncPodStep(devbox, recLabels),
		r.syncPodReadyConditionStep(devbox, recLabels),
	}
}
```

每个 step 自带 `skip` 判断、成功/失败日志、对应的 Condition 类型以及 `EventRecorder` 事件定义，`runSyncPipeline` 顺序跑下来，哪一步挂了直接把具体错误写进对应 Condition 并抛出 Warning Event，排查的时候看一眼 CR 的 `status.conditions` 就清楚卡在哪一步。

顺带一提里面的 `syncKubeAccessStep`：有些用户在 DevBox 里本身就是要写 Operator 或调 `kubectl` 管理自己 namespace 下的资源。开启 `spec.kubeAccess.enabled` 后，Controller 会按 `view / edit / admin` 角色模板在用户 namespace 下自动建好专属的 ServiceAccount、Role/RoleBinding，并通过 Projected Volume 把受限 Token 挂进容器，省去了前端再跑一套 RBAC 初始化 Job 的麻烦。

## 发版流程（DevBoxRelease）与多仓库凭据管理

用户把开发环境调好之后，经常需要导出成一个正式版本镜像拿去部署。这块单独拆了 `DevBoxRelease` CRD 和 `DevboxreleaseReconciler`。

发版有个前提：被导出的 DevBox 必须已经停稳（`Stopped`），确保最后一次改动已经通过 `commit-worker` 落成了内部仓库里的 `sourceImage`。如果用户刚点完发版、DevBox 还在停机提交的过程中，`DevboxreleaseReconciler` 检测到 `!isDevboxReleaseReady(devbox)` 会直接返回，靠 watch `Devbox` 的状态变化事件驱动下一次重入，而不是在队列里空转报错。

真正执行发版时，并不需要把镜像拉下来重新打包，Controller 直接走 Registry HTTP V2 接口把 `status.sourceImage` 的 manifest retag 到目标版本标签即可；遇到源 manifest 不存在这种不可恢复错误直接置为 `Failed` 终止重试，避免无限刷接口。

这里踩过一个多环境部署的坑：线上既有集群内部的私有仓库（可能是自签证书甚至 plain HTTP，返回 Bearer challenge 的行为也不太标准），又有外部的 `ghcr.io` 或客户自建 Harbor。如果全局只配死一套账密和 `--registry-insecure`，要么连不上外部 HTTPS 仓库，要么把内部仓库的凭据泄露给无关域名。

后来在 `internal/registryconfig` 里做成了按规范化 host（含非默认端口）精确匹配的凭据集合：

- 主仓库参数负责生成 commit 镜像地址；
- 额外仓库通过 Secret 挂载 `registries.yaml` 注入；
- 匹配不到配置里的域名时，退回到节点已有的 Docker credential store 或匿名访问，**绝不把主仓库的账号密码发给其他域名**；
- 在 Sealos 部署脚本里，安装器会直接读取节点上的 `/etc/image-cri-shim.yaml` 自动生成默认仓库列表并注入 Secret，Secret 内容变更时滚动 Controller 和 `commit-worker`。

这套拆法跑下来最大的感受是：凡是涉及宿主机本地状态的操作，尽量不要让中心控制面硬伸手过去管，把状态契约定义在 CRD Status 里，让全局 Reconciler 和节点 DaemonSet 各司其职，边界会干净很多。
