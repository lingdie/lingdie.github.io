---
title: DevBox 总结（三）：SSHGate 与 HTTPGate 接入网关设计
date: 2025-09-09 22:05:00
tags:
  - ssh
  - rust
  - pingora
  - kubernetes
  - devbox
categories:
  - Cloud Native
---

云端开发环境建起来之后，外部流量进容器主要有两条路：一条是开发者的 IDE（VS Code Remote、Cursor、JetBrains Gateway）或者终端连 SSH；另一条是用户在容器里跑了个 Web 服务或者 gRPC 服务，想直接通过浏览器或客户端调试端口。

早期最省事的做法是给每个 DevBox 建一个 NodePort Service，SSH 走一个端口，业务端口再开别的。但集群规模一上来问题就很明显：NodePort 默认端口池（30000-32767）就两千多个，实例一多连 paused 状态的机器都占着坑；而且节点公网暴露面太大，防火墙和安全组很难收敛。另外，如果每个临时预览端口都去建一套 Service + Ingress，Ingress Controller 频繁 reload 配置的开销也不小。

所以在 v2 里我们把这两块收敛成了两个专门的网关：`sshgate`（Go 写的四层/SSH 协议网关）和 `httpgate`（Rust 基于 Pingora 写的七层动态路由网关），统一直接转发到后端的 Pod IP。

## SSHGate：共用单端口的公钥路由网关

TCP/TLS 可以靠 SNI 区分域名共用 443 端口，但原生 SSH 协议握手阶段根本没有类似 Host 头的字段。想让所有用户都连同一个入口地址和端口（比如 `ssh devbox@ssh.example.com -p 22`），就得想办法在握手阶段认出“当前连上来的是哪个 DevBox 实例”。

### 1. 靠客户端公钥反查实例与自定义密钥回退

Controller 在初始化每个 DevBox 时，都会生成一对专属的 SSH 密钥存进同名 Secret（`SEALOS_DEVBOX_PUBLIC_KEY` 和 `SEALOS_DEVBOX_PRIVATE_KEY`），用户下载的 SSH 配置里默认就带着这把私钥。

`sshgate` 利用 client-go Informer 同时监听集群里带 `app.kubernetes.io/part-of=devbox` 标签的 Secret 和 Pod，在内存里维护两张表：

- `publicKey.Marshal() -> namespace/devboxName`
- `namespace/devboxName -> DevboxInfo{PodIP, PublicKey, PrivateKey}`

客户端连上来触发 `PublicKeyCallback` 时，网关直接拿对方递过来的公钥去第一张表里查，命中了就拿到目标 Pod IP，再用 Secret 里存的那把平台私钥作为客户端去连后端 Pod 的 22 端口，然后双向转发所有 SSH Channel 和 Global Request。对用户来说，用户名填什么都行，体验跟直连一台独立机器没区别。

不过光靠平台生成的密钥还不够——不少开发者习惯用自己本机现成的 `~/.ssh/id_ed25519.pub`（自己把公钥塞进了容器的 `authorized_keys`）。这种情况下网关内存表里查不到他的个人公钥，`PublicKeyCallback` 就会退回到第二种模式：解析登录用户名（格式编码了短 namespace 和 devbox 名，例如 `devbox@ns-mybox`）定位目标实例，然后走下面的 Agent Forwarding 或 ProxyJump。

### 2. 多副本一致的确定性 HostKey

`sshgate` 线上是 Deployment 多副本挂在负载均衡后面的。如果每个副本启动时随机生成自己的 Server HostKey，客户端每次重连落到不同 Pod 上都会弹那个经典的 `REMOTE HOST IDENTIFICATION HAS CHANGED!` 警告直接拒绝连接。

为了省掉额外挂载和轮转证书文件的麻烦，`hostkey` 模块直接拿环境变量里的 `SSH_HOST_KEY_SEED` 做一次 `SHA-256` 得到 32 字节摘要，喂给 `ed25519.NewKeyFromSeed` 生成确定性私钥。只要同集群配置的 seed 一样，任意扩缩容出来的网关副本指纹完全一致。

### 3. Agent Forwarding 与 ProxyJump 的协议细节

支持用户自带密钥时，网关手里是没有用户私钥的，没法替他向后端 `sshd` 签名认证，只能做跳板转发。这里实现了两种方式：

第一种是 **SSH Agent Forwarding**。这里有个 OpenSSH 协议实现上的坑：按照规范，`auth-agent-req@openssh.com` 不是连接级的请求，而是打开 `session` Channel 之后发过来的 **Channel Request**，并且客户端往往会紧跟着连发 `pty-req`、`env`、`shell` 或 `exec` 请求。但这时候网关还没拿到 Agent 通道，连后端 Pod 的连接都还没建起来。

如果在建好后端连接前把这些请求丢了，终端就会卡死或者丢环境变量。所以在 `handleSessionRequests` 里专门做了一段前置缓冲：先接住客户端的 session channel，在一个带超时的循环里拦截 `auth-agent-req@openssh.com` 主动向客户端打开 `auth-agent@openssh.com` 通道拿来拨通后端 Pod，同时把这期间收到的非 agent 请求（最多缓存 6 个）暂存下来，等后端 session channel 打开后再按原顺序回放过去。

第二种是 **ProxyJump（`direct-tcpip`）**。用户配了 `-J` 跳板机时，客户端会发来 `direct-tcpip` 类型的 Channel。为了防止有人拿网关当通用内网跳板去扫集群里的其他 IP，`handleProxyJumpMode` 解析完请求报文后**直接忽略客户端声明的 `HostToConnect` 和 `PortToConnect`**，强制只往当前认证绑定的那个 `PodIP:22` 拨号建隧道。

另外，双向拷贝 Channel 数据时如果一端刚打完输出就关连接，很容易把 `exit-status` 请求丢在半路，导致远端跑自动化脚本拿不到退出码，这块也专门做了同步等待收尾。

## HTTPGate：用 Rust + Pingora 做无外部存储的动态反代

业务端口预览和容器内 Agent 通信走的是七层网关 `httpgate`，这块用 Rust 和 Cloudflare 开源的 `pingora` 写的，核心诉求就两个：内存占用足够省，以及同时吃下普通 HTTP/1.1（含 WebSocket 升级）和明文 gRPC（h2c）。

### 1. 双层 DashMap 索引与 PartialPod 裁剪

`httpgate` 不依赖 Redis 或数据库，路由信息全部来自 K8s Watch。域名规则定死为：

```text
devbox-<uniqueID>-<port>.<domain>
```

其中 `uniqueID` 是 Controller 初始化 DevBox 时生成的一串随机词（存放在 `status.network.uniqueID`），`port` 可以是具体端口号（如 `8080`），也可以是特殊关键字 `agent`（自动映射到配置的容器内 agent 端口）。

为了不让 CRD 状态变更和 Pod 重建互相耦合，`DevboxRegistry` 里拆成了两个完全独立的 `DashMap`，分别由两个后台 Tokio task 维护：

- `DevboxWatcher`：监听 `Devbox` CRD，维护 `uniqueID -> (namespace, devbox_name)`；
- `PodWatcher`：监听带 `app.kubernetes.io/part-of=devbox` 标签的 Pod，从 `ownerReferences` 里取出所属 DevBox 名字，维护 `namespace/devbox_name -> pod_ip`。

这里做了一个很管用的内存优化。集群里一个完整 `corev1.Pod` JSON 非常大（环境变量、挂载卷、容器状态、managedFields 全在里面），而网关路由其实只需要 `metadata.name`、`metadata.namespace`、`metadata.ownerReferences` 和 `status.podIP` 这几个字段。我们在 `partial.rs` 里自定义了 `PartialPod` 结构体并手动实现 `kube::api::Resource`，反序列化时直接丢掉整个 `spec` 和无关 `status`，加上初始 List 设了分页（`page_size(100)`），常驻内存能省掉七八成，启动瞬间也不会冲高内存峰值。

请求进来时走两步查表，把“没这个实例”和“实例停机了”明确区分开：

- 第一张表没查到 `uniqueID` -> 直接回 `404 devbox not found`；
- 第一张表有、但第二张表没 `PodIP`（比如处于 `Paused` 或 `Stopped`） -> 回 `503 devbox not running`。

### 2. 同时转发 HTTP/1.1 与明文 gRPC（h2c）

除了普通的网页预览，部分内部工具链会通过 `<uniqueID>-agent.<domain>` 跟容器里的服务走 gRPC 通信。Ingress 卸掉外层 TLS 后，打到 `httpgate` 上的流量是明文的。

Pingora 默认监听口只按 HTTP/1.1 处理明文连接，所以启动服务时要显式把 Server 的 `HttpServerOptions.h2c` 打开。到了 `request_filter` 阶段：

1. 优先从 HTTP/1.1 的 `Host` 头取域名；如果是 HTTP/2 请求没有 `Host` 头，就回退读 `:authority` 伪头部。
2. 判断如果当前连接版本是 `Version::HTTP_2` 且 `content-type` 以 `application/grpc` 开头，就在请求上下文里标为 `UpstreamProtocol::Grpc`。
3. 在 `upstream_peer` 构造上游 `HttpPeer` 时，一旦识别为 `Grpc`，把 `peer.options.alpn` 设成 `ALPN::H2`，让 Pingora 用明文 HTTP/2 去连后端 Pod IP；其余请求一律保持 HTTP/1.1。

另外 Pingora 默认在连不上上游时只会吐一个空 body 的 502，前端或调试者很难分清是外层 Ingress 挂了还是容器里没监听那个端口，所以我们重写了 `fail_to_proxy`，按 `ErrorSource::Upstream / Downstream / Internal` 分类写回带明确原因的响应体（客户端自己断开的读写错误则直接置 0 静默收掉）。
