---
title: DevBox 总结（二）：基于 LVM Thin Pool 的 Containerd Snapshotter 与镜像提交方案
date: 2025-08-21 20:42:00
tags:
  - containerd
  - snapshotter
  - lvm
  - estargz
  - devbox
categories:
  - Cloud Native
---

上一篇聊了 DevBox 的 Controller 怎么和节点上的 `commit-worker` 配合，这篇往下扎一层，聊聊宿主机上的 containerd、`devbox` snapshotter、容器读写层 commit 以及 RW 层容量监控是怎么做出来的。

## 为什么原生 overlayfs 不够用

默认 containerd 用的 `overlayfs` snapshotter 把容器读写层（`upperdir`）直接建在宿主机根盘或数据盘的普通目录上，放在多租户云开发环境里主要有三个问题：

1. **容量限制不好控**：不做文件系统 project quota 的话，单个用户在容器里 `dd` 一个大文件就能把整台节点的 containerd 分区打满；而 K8s 原生的 `ephemeral-storage` 靠周期扫描目录驱逐 Pod，反应慢且粗暴，驱逐完读写层也没了。
2. **Pod 重建后读写层解耦难**：标准 CRI 的容器生命周期跟 snapshot 强绑定，Pod 一删，对应的 active snapshot 就被清掉了，没法做到“Paused 删掉 Pod 释放 CPU/内存，但把这块读写层原封不动留在节点上等下次挂载”，也没法在 Pod 退出后再单独挂起这块读写层慢慢做 commit。
3. **反复停启导致镜像层无限叠高**：如果每次 `Stopped` 都往原镜像顶上追加一层 diff，用户停启几十次以后镜像层数就炸了，甚至撞上 overlayfs 的最大层数限制（128 层），历史垃圾文件也一直占着体积。

针对这几个痛点，我们在定制版的 `labring/containerd` 里加了 `devbox` snapshotter（基于 overlay 改造，底层接 LVM Thin Pool），同时改了 `labring/nerdctl` 的 commit 逻辑来配合。

## Devbox Snapshotter 的存储与挂载原理

节点的 containerd 配置里会注册一套独立的 runtime handler `devbox-runc`，绑定到 `devbox` snapshotter：

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.devbox-runc]
  runtime_type = "io.containerd.runc.v2"
  snapshotter = "devbox"

[plugins."io.containerd.snapshotter.v1.devbox"]
  root_path = "/var/lib/containerd/io.containerd.snapshotter.v1.devbox"
  upperdir_label = true
  sync_remove = false
  lvm_vg_name = "devbox-vg"
  thin_pool_name = "devbox-vg-thinpool"
```

K8s 侧通过 `RuntimeClass`（默认 `devbox-runtime`）把 DevBox Pod 路由到 `devbox-runc`。普通镜像只读层（lowerdir）依然走普通目录解压，只有带了 DevBox 特定 snapshot label 的容器可写层，才会触发 LVM Thin Volume 逻辑。

### 用 BoltDB 把 contentID 和 LV 绑在一起

Controller 创建 Pod 时，会在注解上带 `devbox.sealos.io/content-id` 和 `devbox.sealos.io/storage-limit`，传到 snapshotter 时对应下面几个 label：

- `containerd.io/snapshot/devbox-content-id`
- `containerd.io/snapshot/devbox-storage-limit`
- `containerd.io/snapshot/devbox-init`

`snapshots/devbox` 在原有的 BoltDB 元数据存储里多开了一个 `DevboxStoragePathBucket`，专门维护 `contentID -> {lvName, snapshotKey, path, status}` 的映射。

当 `Prepare`（即 `createSnapshot`）被调用且带有 `content-id` 和 `storage-limit` 时，分两种情况走：

**1. 首次创建该 `contentID`**：
在配置的 `lvm_vg_name / thin_pool_name` 里切出一块名为 `devbox-<contentID>` 的 thin LV，大小按 `storage-limit` 设定（天然硬配额，写满只会让这个容器自己报 `no space left on device`，不会冲垮宿主机），格式化成 `ext4`，挂载到当前 snapshot 目录（`snapshots/<id>`），并在里面建好 overlay 需要的 `fs`（即 `upperdir`）和 `work` 目录。如果中间任何一步出错，`defer` 里会自动 `umount` 并强删这块 LV，避免残留孤儿卷。

**2. 复用已有的 `contentID`（比如 Paused 后恢复，或者停机后拉起影子容器做 Commit）**：
查 BoltDB 发现这个 `contentID` 底下已经有 `lvName` 了，就不再新建 LV。先调 `resizeLVMVolume` 看配额有没有调大，然后更新 BoltDB 里的关联 `snapshotKey` 和挂载路径，直接把原来的 `/dev/<vg>/devbox-<contentID>` 挂载到新 snapshot 的目录上。之前写进 `fs/` 里的数据就原样接上了。

这里有个很容易踩的细节：**同一个 LV 同一时刻不能挂在两个 active snapshot 下**。所以在旧容器退掉、新容器（或影子 commit 容器）创建之前，需要先通过 snapshotter 的 `Update` 接口给旧 snapshot 打上 `containerd.io/snapshot/devbox-unmount-lvm = "true"` 标签。snapshotter 收到这个更新后，会在写事务里把 BoltDB 中绑定的旧 `snapshotKey` 清掉，并调 `syscall.Unmount` 卸载旧挂载点，把这块 LV 释放出来给下一个消费者挂。

## 停机 Commit 流程与层合并优化

Pod 被删掉之后，CRI 建的那个容器和 sandbox 其实已经没了，那 `commit-worker` 怎么把留在 LV 里的数据打成镜像？

做法是在 `commit.go` 里利用 `nerdctl` 的包能力，在 `k8s.io` 命名空间下**创建一个不启动的影子容器（CreateContainer without Task）**：

- 指定相同的 `baseImage`、`contentID` 和 `storageLimit`；
- `InRun: false`、`Restart: "no"`、`NetworkMode: "none"`（特别注意不能注册进 containerd 的 restart monitor，否则它会一直轮询一个根本不存在的 task）；
- 容器创建过程中，snapshotter 看到相同的 `contentID`，就会把刚才解绑出来的那块 LV 挂到这个影子容器的 snapshot 上；
- 接着直接对这个影子容器执行 `container.Commit`，把 `upperdir`（也就是 LV 里的 `fs` 目录）跟 `parent` 做 diff 导出成新层，生成镜像后推往内部 Registry。

这里面做了几处专门针对生产环境的改造：

### 1. 避免层数无限叠加（mergeBaseImageTopLayer）

如果用户每次停机都往镜像上叠一层，用久了层数一定会失控。我们在 CRD 里加了 `mergeBaseImageTopLayer` 开关，两头配合把“上一次 commit 出来的用户层”和“这一次的新改动”拍平：

- **启动挂载阶段**：当这个实例已经 commit 过（带有 `devbox-init` 标记）时，snapshotter 在 `createSnapshot` 里会把新 snapshot 的 `directParent` 往上提一层（跳过原镜像的最顶层），然后把原镜像顶层目录里的文件直接拷贝进新建的 LV `fs/` 目录里。
- **停机 Commit 阶段**：在定制版 `labring/nerdctl` 的 `Commit` 实现里加了 `RemoveBaseImageTopLayer` 选项。生成新镜像的 `ImageConfig` 和 `Manifest` 时，直接裁掉 `baseConfig.RootFS.DiffIDs`、`History` 以及 `baseMfst.Layers` 的最后一项，再把当前 LV 导出的完整 diff 层追加上去。

这样不管用户反复停启多少次，基础环境层保持不变，顶上永远只有一层最新的用户工作区数据。

### 2. Commit Lease 与 GC 保护

大体积开发环境做 diff 导出（尤其是开了压缩或 eStargz 转换）耗时不短。如果中途正好撞上 containerd 的垃圾回收，还没写进最终 manifest 引用树的临时 blob 很容易被 GC 清掉。所以在 `nerdctl` 的 `Commit` 入口显式加了带过期时间的 `client.WithLease`（`CommitLeaseDuration` 可配），并在写完新 manifest 和 config blob 时完整打好 `containerd.io/gc.ref.content.*` 和 `containerd.io/gc.ref.snapshot.<sn>` 标签，保证整个链路不断引用。

### 3. Stargz / eStargz 加速支持

除了默认的 `devbox` snapshotter，Controller 也兼容了 `devbox-stargz-runtime`。用 eStargz 跑按需拉取时，要求基础镜像本身是 eStargz 格式。我们在 `ensureBaseImageForSnapshotter` 里加了一道检查：拉下基础镜像后先扫一遍 manifest layers 里有没有 `containerd.io/snapshot/stargz/toc.digest` 注解；如果有普通层，就在本机用 `estargzconvert` 原地转成 eStargz（已转换的层直接跳过），并且外层包了一个 `singleflight.Group`，防止同一台节点上同时拉起多个同模板 DevBox 时重复做镜像转换把 CPU 打满。

## 异步回收 LV 与节点 Patch 交付

LV 的清理走的是异步标记回收（配置里 `sync_remove = false`，对应 `AsynchronousRemove`）：

当 DevBox 进入 `Stopped`（已推完镜像）或者被彻底删除时，`commit-worker` 会调 `SetLvRemovable`，给对应 snapshot 打上 `containerd.io/snapshot/devbox-remove-content-id = <contentID>` 标签，把 BoltDB 里的状态标成 `Removed`，再删掉容器和临时镜像。真正耗时的目录清理、检查 `/proc/mounts` 兜底卸载（同时处理 `/dev/mapper/...` 软链解析）以及 `lvremove` 销毁逻辑卷，统一收口在 snapshotter 的 `Cleanup` 周期任务里做，避免阻塞主流程。

至于这套定制版 containerd 怎么装到存量 K8s 节点上，我们在仓库里维护了 `containerd/patch` 镜像：

- 启动前先校验宿主机 VG 和 LV 的 `segtype` 确实是 `thin-pool`（脚本只做校验，绝不擅自格式化或改客户的 LVM 结构）；
- 把原生的 `/usr/bin/containerd` 和 `/etc/containerd/config.toml` 备份到 `/var/lib/devbox-containerd-patch/original`；
- 用 `awk` 幂等剥掉旧的 devbox 配置块再重新渲染写入，只有 `cmp -s` 发现二进制或配置真变了才重启 `containerd -> kubelet -> image-cri-shim`；
- 调 `ctr plugins ls` 确认 `io.containerd.snapshotter.v1 devbox` 状态是 `ok` 后，再给节点打上 `devbox.sealos.io/node` 标签和可选的 `NoSchedule` 污点；卸载时反向恢复备份即可。

## 节点本地采集 RW 层容量

有了 `storage-limit` 硬配额之后，用户在界面上得能看到自己的读写层用了多少空间，不然磁盘写满了连 SSH 都登不上去还不知道原因。

这块数据在节点本地，如果让 `commit-worker` 周期性扫完写回 `Devbox.Status`，几百个实例每 30 秒刷一次，APIServer、etcd 还有 watch `Devbox` 的所有组件都会被无意义的 `resourceVersion` 变更淹没。另外我们也不想在业务容器外头直接跑 `lvs` 命令去解析输出。

最后的做法是在 `commit-worker` 进程内挂了一个 `RWStorageCollector`，复用它的 `/metrics` 端口通过 `VMServiceScrape` 接进 VictoriaMetrics：

1. 采集器只筛 `commitRecord.Node == NODE_NAME` 的本机 DevBox，并且对外暴露的 Prometheus 指标只带 `namespace / devbox / node / snapshotter` 四个标签，**故意不带 `contentID`**，防止每次停启切换 `contentID` 造成时间序列高基数膨胀。
2. 对 `spec.state == Running` 的实例，通过 `content-id` label 找到对应 snapshot，直接调 containerd 标准接口 `snapshotService.Usage(ctx, snapshotName)` 拿字节数（底层由 snapshotter 对 active snapshot 的 `upperPath` 做 `fs.DiskUsage`），同时把 `devbox_rw_storage_collect_active` 置为 `1`。
3. 一旦 `spec.state` 切到 `Paused / Stopped / Shutdown`，立刻停止对它的 `Walk` 和 `Usage` 调用（否则停机清理 LV 的窗口期会报一堆 `snapshot_not_found` 假错误），但只要当前 `contentRecord` 还在本节点，内存里会**保留最后一次成功采集到的 `used_bytes` 和 `usage_ratio`**，仅把 `collect_active` 置为 `0`。前端看到 `collect_active=0` 就知道这是停机前最后一次水位，体验上不会一停机监控图就断成空白。
