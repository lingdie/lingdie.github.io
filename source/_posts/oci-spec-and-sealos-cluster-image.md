---
title: OCI 规范入门与 Sealos 集群镜像实现原理
date: 2024-10-16 21:20:00
tags:
  - oci
  - sealos
  - kubernetes
  - container
categories:
  - Cloud Native
---

使用 Docker、containerd 时都会接触到 OCI（Open Container Initiative）。常见理解是「镜像对应一个容器运行环境」，但 Image Spec 与 Distribution Spec 并不要求镜像内容必须是可执行进程。Sealos 的集群镜像沿用同一套 OCI 格式，将 Kubernetes 相关物料——宿主机二进制、配置模板、Helm Chart，以及依赖的业务容器镜像——一并打包为可分发制品。

下文先说明与此相关的 OCI 结构，再结合 `sealos/lifecycle` 的实现说明集群镜像的构建与运行过程。

## OCI 相关概念

OCI 包含三份规范：

- Image Spec：镜像在磁盘上的组成与元数据格式
- Distribution Spec：客户端与 Registry 的拉取、推送协议（沿用 Docker Registry HTTP API V2）
- Runtime Spec：`runc` 等运行时在拿到 rootfs 与 `config.json` 后如何创建进程

集群镜像主要依赖前两份。由于集群镜像不会作为普通容器由 `runc` 启动，Runtime Spec 基本不参与这条链路。

### 镜像对象结构

Registry 以内容寻址方式存储对象，键为 `sha256` digest，对象统称为 Blob。关联关系由下列结构描述：

**Image Index**（多架构时使用）  
媒体类型为 `application/vnd.oci.image.index.v1+json`。例如拉取 `kubernetes:v1.29.0` 时，Index 中记录 `linux/amd64`、`linux/arm64` 各自对应的 Manifest digest，客户端按本机架构选择其一。

**Image Manifest**  
单个架构的清单。本身不包含 Env、Cmd，作用是引用一份 Config，并按顺序列出各 Layer 的 `mediaType`、`digest`、`size`。

**Image Configuration**  
对应 `docker inspect` 中的大部分字段，主要包括：

- `config`：`Env`、`Entrypoint`、`Cmd`、`Labels`、`WorkingDir` 等
- `rootfs.diff_ids`：各层解压后 tar 的 sha256（Manifest 中记录的是压缩后 digest）
- `history`：构建历史

**Filesystem Layers**  
`.tar.gz` 或 `.tar.zst` 包，按顺序叠加；删除通过 `.wh.<filename>`（whiteout）表示，叠加结果即为 rootfs。

若不调用 `runc`，Image Spec 实质上是带 Labels / Env / Cmd 的分层归档，并可复用现有 Registry 基础设施。Helm Chart 推送到 OCI、Wasm 制品 OCI 化，以及 Sealos 集群镜像，均基于这一能力。

## Sealos 集群镜像

离线部署 Kubernetes 的难点通常不在 `kubeadm init` 本身，而在物料组织：kubelet、containerd 等二进制，内核与系统配置脚本，CNI 与组件 YAML、Charts，以及大量业务镜像。传统方式多为大型 tar 包配合 Ansible，版本或组件变更时往往需要整包重建。

`lifecycle` 基于 `containers/storage` 与 `buildah`，通过 OCI Config 中的 `sealos.io.type` 将镜像分为三类：

- `rootfs`：集群底座，例如 `labring/kubernetes:v1.29.0`，包含 kubeadm、kubelet、kubectl、containerd、image-cri-shim、sealctl，以及 systemd unit 与初始配置。集群必须先应用 `rootfs`。
- `patch`：对底座的增量补丁，与 `rootfs` 一样同步到全部节点。
- `application`：网络插件、Ingress、业务 Chart 等。默认仅分发到 `master0` 执行 apply 或 helm。

### 构建过程

`Kubefile` / `Sealfile` 语法与 Dockerfile 兼容，`pkg/buildah/build.go` 调用 `imagebuildah.BuildDockerfiles` 生成标准 OCI 镜像。额外步骤由 `runSaveImages`（`pkg/buildah/imagesaver.go`）完成：

1. 扫描构建上下文中的镜像引用：
   - `charts/`：Helm 渲染结果中的镜像
   - `manifests/`：YAML 与 `.tmpl` 中的 `image:`
   - `images/shim/`：显式镜像列表，支持 `.sealignore`；`images/skopeo/` 可导入 `docker-archive` / `oci-archive`
2. 不采用对每个镜像单独 `docker save` 的方式（公共层难以去重）。`SaveImages` 在本机 `127.0.0.1` 启动临时 Distribution Registry，将存储根目录指向上下文中的 `registry/`，再按目标 `platform` 将依赖镜像并发推入该目录。
3. 落盘完成后，buildah 继续执行 `COPY` 等指令，将二进制、YAML、Charts 与已形成 `/v2/blobs/sha256/...` 结构的 `registry/` 一并写入镜像层。

结果仍是合规 OCI 镜像，内部 Blob 可去重，并可 `sealos push` 至 Docker Hub、GHCR、Harbor 等仓库。

若需合并多个镜像，`pkg/image/merge.go` 中的 `MergeDockerfileFromImages` 读取各源镜像 Config，合并 Labels、Env、Entrypoint、Cmd，生成以 `FROM scratch` 开头、包含多条 `COPY --from=...` 的 Containerfile 再构建。

### 运行过程

执行 `sealos run labring/kubernetes:v1.29.0 labring/cilium:v1.14.1 --masters ... --nodes ...` 时，集群镜像不会作为容器进程启动。编排由 `CreateProcessor`（`pkg/apply/processor/create.go`）完成：

```text
Check -> PreProcess -> RunConfig -> MountRootfs -> MirrorRegistry -> Bootstrap -> Init -> Join -> RunGuest
```

与 OCI 关联较紧的步骤如下。

**PreProcess**  
`MountClusterImages` 先 `buildah.Pull`，再 `buildah.Create` 获取 Overlay 挂载点。`OCIToImageMount` 读取 OCI Config：

- Labels 中的 `sealos.io.type`（rootfs / patch / application）与 K8s 版本
- Env，并与 Clusterfile、命令行 `--env` 合并
- Entrypoint / Cmd（去除 `/bin/sh -c` 包装），供后续 RunGuest 使用

原先面向 Runtime 的配置字段，在此作为安装参数与入口命令使用。

**MountRootfs**  
`pkg/filesystem/rootfs/rootfs_default.go` 负责将文件分发到目标主机：

- rootfs / patch：通过 SSH 同步除 `registry/` 以外的内容到各节点（例如 `/var/lib/sealos/data/default/rootfs`），再执行 `sealctl render` 渲染 `.tmpl`
- application：仅同步到 master0 的应用工作目录（`.../applications/<name>/workdir`）

**MirrorRegistry**  
构建阶段写入 `registry/` 的业务镜像，在此同步到集群仓库节点（`pkg/filesystem/registry/sync.go`）。

直接整目录拷贝会导致无关 Blob 的重复传输。`Sync` 优先在目标节点执行 `sealctl registry serve`，临时启动监听 `5050` 的 HTTP Registry（数据目录为 `/var/lib/registry`），管理机再通过 `containers/image` 的 copy 协议做增量同步；若无法启动临时服务，则回退为 SSH 目录同步。采用标准 Distribution 协议时，目标端已有 Blob 可在 HEAD 阶段跳过。

**image-cri-shim**  
`kubeadm init` 与业务 YAML 中的镜像名仍可能是外网地址（如 `registry.k8s.io/pause:3.9`），而实际数据位于 `sealos.hub:5000`。逐一改写 Chart 与 YAML 中的域名成本高且易遗漏。

节点侧部署 `image-cri-shim`：kubelet 的 CRI endpoint 先连接 shim，再转发至 containerd。处理 `PullImage`、`ImageStatus` 等请求时，shim 用原始镜像名查询私有仓库（带 LRU 缓存），命中后改写为内部地址并注入认证，再交给 containerd。上层清单无需修改，在线与离线可共用同一集群镜像。

**RunGuest**  
在 Bootstrap、Init、Join（含 lvscare 静态 Pod 提供本机 IPVS）完成后，按挂载顺序执行：

- rootfs / patch：在全部节点并发执行 Config 中的 Entrypoint 与 Cmd
- application：仅在 master0、进入对应工作目录，注入合并后的 Env，再执行 Entrypoint 与 Cmd（如 `helm upgrade`、`kubectl apply`）

## 最小示例

将带 YAML 的组件打包为 application 集群镜像，目录可如下：

```text
my-app/
├── Kubefile
└── manifests/
    └── deploy.yaml    # 使用 ghcr.io/xxx/app:v1.0.0 等镜像地址
```

`Kubefile`：

```dockerfile
FROM scratch
ENV REPLICAS="2"
COPY manifests ./manifests
CMD ["kubectl apply -f manifests/deploy.yaml"]
```

在可访问外网的构建机上：

```bash
sealos build -t hub.example.com/infra/my-app:v1.0.0 -f Kubefile .
```

构建过程会解析 YAML 中的业务镜像，按 Registry Blob 格式写入镜像内 `registry/`，并将 `ENV`、`CMD` 写入 OCI Config。

将镜像分发到目标环境后：

```bash
sealos run hub.example.com/infra/my-app:v1.0.0
```

业务镜像同步至集群内部仓库，并在 master0 执行安装命令。整条链路未引入新的打包格式，而是复用 OCI 的分层文件系统、Config 元数据与 Distribution 协议。
