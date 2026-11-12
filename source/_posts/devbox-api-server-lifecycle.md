---
title: DevBox 总结（四）：API Server、容器内命令代理与生命周期调度
date: 2025-10-27 19:53:00
tags:
  - golang
  - kubernetes
  - reverse-proxy
  - devbox
categories:
  - Cloud Native
---

除了面向浏览器用户的 Next.js 控制台，DevBox 还需要给外部系统、自动化脚本以及 AI Agent（比如类 Codex 的远程沙箱调用场景）提供一套干净的编程接口：一条请求拉起环境、执行命令、上传下载文件、按路径反代访问里面的服务，用完了到点自动暂停甚至归档释放。

这块独立做成了 Go 服务 `server`（即 `devbox-api`）。如果只是把 K8s CRD 包装成 CRUD 其实没什么好说的，实际做下来真正花心思的主要是三件事：怎么绕开高开销的 `kubectl exec`、怎么做无 Redis 依赖的子路径反向代理，以及怎么实现轻量的定时休眠/归档调度。

## 绕开 APIServer Exec，直连 Pod 内 SDK Server

外部 Agent 调沙箱环境时，`POST /exec`、文件上传和文件下载是非常高频的操作。

最开始如果图省事，直接调 K8s APIServer 的 `/pods/{name}/exec`（SPDY 或 WebSocket 升级，甚至在容器里拼 `tar` 管道传文件），并发稍微高一点问题就全暴露出来了：每次执行命令都要经过 `server -> kube-apiserver -> kubelet -> containerd` 一长串链路，APIServer 的连接开销和审计日志压力很大，大文件走 exec 流也很容易超时断流，而且很难精确拿到标准输出、标准错误和退出码的结构化结果。

所以在 v2 里，每个 DevBox 容器内部都会跑一个轻量的守护进程 `devbox-sdk-server`（监听 `9757` 端口），`server` 在集群内直接走 Pod IP 跟它通 HTTP：

1. **鉴权隔离**：每个 DevBox 初始化时，Secret 里除了 SSH 密钥还会存一份专属的 `SEALOS_DEVBOX_JWT_SECRET`（内存带 5 分钟 TTL 缓存）。`server` 校验完外部调用方的租户 JWT（从中提取绑定的 `namespace`，不允许请求体越权指定 namespace）后，再用目标 DevBox 自己的密钥签发短期 Bearer Token 去调 `http://<podIP>:9757`。
2. **冷启动探活等待**：刚恢复或刚拉起的 Pod，有时候 K8s 已经标成 `Running` 了，但里面的 `9757` 端口还没监听好。`resolveRunningPodAndSDKServer` 拿到 Pod IP 后，会先用 250ms 短连接超时、100ms 间隔做一个最多 3 秒的 `waitForSDKServerReachable` 探测，平滑吃掉容器刚启动那一两秒的抖动，避免调用方刚看到 Running 一调 `exec` 就吃 502。
3. **Stdin 兼容与超时透传**：同步命令执行走 `/api/v1/process/exec-sync`，结构化返回 `stdout`、`stderr` 和 `exitCode`；当调用方传了非空 `stdin` 时，自动包装命令并通过环境变量安全注入输入流，保证老接口语义不变。

## 双监听口与基于内存索引的子路径网关

在对接外部平台时，很多环境不方便配泛域名证书（也就是没法给每个沙箱分一个三级子域名），更希望只暴露一个固定域名，靠路径前缀来区分不同实例，比如：

```text
https://devbox-gateway.example.com/codex/<uniqueID>/...
```

为此 `server` 进程同时开了两个监听端口：`8090` 专门接带租户 JWT 鉴权的管控 REST API，`8091` 专门接 `/codex/{uniqueID}` 和 code-server 等应用网关流量，两边可以在外层挂不同的 Ingress 策略。

为了让路径转发不依赖外部 Redis，我们在进程内维护了双映射索引（`uniqueID -> gatewayIndexEntry` 和 `namespace/name -> uniqueID`）：

- 正常情况下通过监听 `Devbox` 资源变化实时同步 `status.network.uniqueID`；
- 请求打到 `/codex/{uniqueID}/...` 时，如果本地内存索引恰好没命中（比如实例刚建出来 Informer 还没推到当前副本），`handleGatewayProxy` 会主动调一次 `refreshGatewayIndex` 做按需兜底刷新，再查不到才回 404，避免多副本短暂缓存不一致导致刚建好的沙箱打不开。
- 转发到 `http://<podIP>:1317` 时，底层连的是 Pod IP，但逻辑 Host 会构造成 `<uniqueID>.<namespace>.svc.cluster.local:1317`，同时处理好路径前缀剥离和上游 `Location` 重定向改写，让跑在子路径下的 Web 服务能正常跳转。

另外在调 `GET /api/v1/devbox/{name}` 查询实例详情时，接口会直接把拼好的网关 URL 和用该实例密钥签好的网关访问 Token 一并返回，外部调用方拿到就能直接连，不需要二次换票。

## 带选主的生命周期调度器（自动暂停与归档）

沙箱场景里很多实例是用完即弃或者闲置一会儿就该停掉的。如果每个实例都让业务方自己维护定时器去调 pause/stop，客户端一挂机器就一直跑着烧钱；所以在创建和续期接口里直接支持了两个策略注解：

- `devbox.sealos.io/pause-at`：到指定时间点自动从 `Running` 切到 `Paused`（释放算力，保留节点本地 LV）；
- `devbox.sealos.io/archive-after-pause-time`：进入 `Paused` 超过指定时长后，自动切到 `Shutdown`（触发节点 commit 镜像并彻底释放本地 LV 归档）。

同时给这类带策略的对象打上 `devbox.sealos.io/lifecycle-scheduled=true` 标签，方便过滤列表，不扫全量无关实例。

### 为什么不用固定高频轮询

如果无脑起一个每 5 秒扫全表的协程，集群里实例多了会产生大量无意义的 List/Patch；但如果把扫描间隔拉长到几十分钟，到点停机的精度又太差。

我们在 `lifecycle.go` 里做成了一个**事件唤醒 + 最近到期单 Timer + 周期全量兜底**的混合模型，外层再用 `coordination.k8s.io/v1` 的 `LeaseLock`（`devbox-api-lifecycle`）做选主，保证多副本部署下同一时刻只有一个副本在跑调度循环：

1. **内存维护 `nextByKey` 到期表**：启动当选 Leader 后先做一次带 label 过滤的全量扫描（`reconcileLifecycleFull`），算出每个实例下一次需要动作的时间点（比如还没到期的 `pauseAt`，或者已经 Paused 实例的 `pausedAt + archiveAfterPause`），存进 `map[ObjectKey]time.Time`。
2. **动态重置单个 `time.Timer`**：每次循环结束时，取整个 map 里最早的那个时间点作为下一次唤醒间隔（下限 1 秒，上限不超过兜底周期 `lifecycleResyncInterval`，默认 30 分钟）。平时没有任何轮询开销，哪个实例到点了 Timer 就精准醒过来只处理到期的 key。
3. **API 写操作即时唤醒**：当用户通过当前副本调了创建、修改 `pauseAt`、手动暂停/恢复或删除接口时，Handler 会往带缓冲的 `lifecycleNotifyCh` 发一个按 key 刷新的信号，调度循环立刻更新这台实例的下次唤醒时间并重设 Timer。
4. **30 分钟全量 Ticker 兜底**：考虑到 API 请求可能落到非 Leader 副本上，或者有人直接用 `kubectl` 改了注解，每隔 `30m` 还会跑一次全量校准，同时自动清理那些已经没有生命周期注解的对象上的 `lifecycle-scheduled` 标签。

在真正改状态的时候，`patchDevboxMetadataAndState` 会显式校验当前状态（比如确认此刻确实还是 `Running` 才改成 `Paused`，同时删掉已消费的 `pause-at`、写上 `paused-at` 时间戳），用 `RetryOnConflict` 防止和用户手动恢复的操作撞车。

回头看整个 DevBox 后端这四块——Controller 拆开全局编排和节点本地提交、Snapshotter 用 LVM Thin Pool 管住配额和快照复用、SSH/HTTP 网关收掉端口暴露面、API Server 把高频命令执行和生命周期托管做轻——核心思路其实一直没变：把 K8s 控制面擅长做的声明式编排留给 APIServer，把高频数据面和宿主机本地存储操作坚决剥离到直连通道和节点本地组件里去。
