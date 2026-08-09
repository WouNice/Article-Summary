# 第6章：容器化技术与 Docker

## 本章学习目标

- 理解容器与虚拟机的本质区别
- 掌握 Docker 的核心概念（镜像、容器、网络、数据卷）
- 熟悉 Dockerfile 编写的最佳实践
- 了解镜像优化和安全扫描的基本方法

## 容器化：环境一致性的终极方案

### 痛点回顾

```
"在我的机器上能跑啊！"
  ↓
开发环境：macOS + Node 18
测试环境：Ubuntu + Node 16
生产环境：CentOS + Node 14

同一个代码，三个环境，三种行为。
```

### 容器解决什么

容器将应用及其**所有依赖**（代码、运行时、系统库、配置文件）打包在一个标准化单元中，确保在任何环境中行为一致。

### 容器 vs 虚拟机

```
┌────────────────┐  ┌────────────────┐
│   App A        │  │   App A        │
│   Libs         │  │   Libs         │
├────────────────┤  ├────────────────┤
│ Guest OS       │  │                 │
├────────────────┤  │   Container    │
│ Hypervisor     │  │   Engine       │
├────────────────┤  ├────────────────┤
│  Host OS       │  │  Host OS       │
├────────────────┤  ├────────────────┤
│  Hardware      │  │  Hardware      │
└────────────────┘  └────────────────┘
   虚拟机模型           容器模型
```

| 维度 | 虚拟机 | 容器 |
|------|--------|------|
| 启动时间 | 分钟级（需启动完整 OS） | 秒级（进程级启动） |
| 资源开销 | 每个 VM 独占 Guest OS | 共享宿主机内核 |
| 镜像大小 | GB ~ 数十 GB | MB ~ 数百 MB |
| 密度 | 单机数台 ~ 数十台 | 单机数十 ~ 数百个 |
| 隔离性 | 强（独立内核） | 中（共享内核） |
| 一致性 | 按 OS 模板 | 标准化镜像 |

## Docker 核心概念

### 镜像（Image）

**镜像**是一个只读的模板，包含运行应用所需的全部文件。

```
┌─ Application Layer ─┐  ← 你的代码
├─ Dependency Layer ──┤  ← npm install / pip install
├─ Config Layer ──────┤  ← 环境变量、配置文件
├─ Base OS Layer ─────┤  ← Ubuntu / Alpine / Debian
└─ Kernel Access ─────┘  ← 宿主内核（不包含在镜像中）
```

镜像由多层（Layer）组成，每层是 Dockerfile 中的一条指令。**层可以共享和缓存。**

```bash
docker pull nginx:latest      # 拉取镜像
docker images                 # 查看本地镜像列表
docker rmi <image>            # 删除镜像
docker history <image>        # 查看镜像的分层历史
```

### 容器（Container）

**容器**是镜像的运行实例，本质是一个进程。

```bash
docker run -d --name my-nginx -p 8080:80 nginx
#         ↑           ↑              ↑
#       后台运行   容器名称    端口映射：宿主机:容器

docker ps                    # 列出运行中的容器
docker stop <container>      # 停止容器
docker start <container>     # 启动已停止的容器
docker rm <container>        # 删除容器
docker exec -it <container> bash  # 进入容器内部
```

### 容器生命周期

```
              ┌──────────┐
   docker     │  Created  │
   create ──→ │ (已创建)  │
              └────┬─────┘
                   │ docker start
                   ▼
              ┌──────────┐
              │  Running  │ ◀──── docker restart
              │ (运行中)  │
              └────┬─────┘
                   │
          ┌────────┼────────┐
          │                │
   docker stop       进程退出
          │                │
          ▼                ▼
    ┌──────────┐    ┌──────────┐
    │ Stopped  │    │ Exited   │
    │ (已停止)  │    │ (已退出)  │
    └────┬─────┘    └────┬─────┘
         │               │
         └───────┬───────┘
                 │ docker rm
                 ▼
            ┌──────────┐
            │ Removed  │
            │ (已删除)  │
            └──────────┘
```

### 镜像仓库（Registry）

```bash
docker pull <registry>/<image>:<tag>
# 默认：docker.io/library/nginx:latest

docker tag my-app:v1 my-registry.com/my-app:v1
docker push my-registry.com/my-app:v1
```

常见的镜像仓库：
- **Docker Hub**：公共仓库，默认 registry
- **Harbor**：企业级私有镜像仓库（开源）
- **AWS ECR / GCR / ACR**：云厂商托管
- **Nexus / Artifactory**：通用制品仓库，也支持 Docker

### 镜像仓库选型对比

不同规模的团队对镜像仓库的需求差异巨大。以下是主流方案的详细对比：

| 对比维度 | Harbor | Docker Hub | AWS ECR | 自建 Registry (Docker Distribution) | Nexus Repository |
|---------|--------|-----------|--------|--------------------------------------|-----------------|
| 部署方式 | Docker Compose / Helm | SaaS 托管（付费高可用） | 托管服务 | Docker 单容器启动 | Java 应用 / Docker |
| 安装复杂度 | 中（需 PostgreSQL + Redis） | 零部署 | 零部署（云 API 创建） | 极低（一行命令） | 中（需 JVM + 存储） |
| 镜像存储 | 本地 / S3 / GCS / Azure | 托管存储 | S3（由 AWS 管理） | 本地文件系统 / S3 | 本地 / S3 / Blob |
| 高可用方案 | Helm 多副本 + 共享存储 | 内置高可用 | 内置高可用 | 需自行搭建负载均衡 | 集群部署 |
| 镜像漏洞扫描 | 内置（Trivy/Clair） | Docker Scout（付费） | 内置（Inspector） | 无（需集成 Trivy） | 内置（Sonatype） |
| 复制/同步 | 支持 Pull-Through 缓存 + 跨地域复制 | 高级订阅 | 跨区域复制（付费） | 无原生支持 | 支持代理远程仓库 |
| RBAC/权限 | 细粒度（项目 + 角色） | 简单（公开/私有） | IAM 细粒度 | 无（基础 HTTP Auth） | 角色 + 权限 |
| Webhook 通知 | 支持 | 支持（付费） | 支持（EventBridge） | 无 | 支持 |
| 自我审计 | 完整操作日志 | 有限 | CloudTrail 审计 | 无日志 | 有日志 |
| 月成本（500GB） | 自建服务器成本 | $7/月起（付费计划） | 存储费 $0.10/GB + 数据传输 | 服务器成本 | 服务器成本 |
| 镜像垃圾回收 | 支持（GC Job） | 自动 | 生命周期策略 | 需手动 GC | 自动策略 |
| 适用场景 | 企业自建、合规要求 | 开源项目、小团队 | AWS 原生环境、无自建运维 | 开发测试环境 | 已有 Nexus 生态 |

#### 选型建议

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 开源项目 / 个人 | Docker Hub 免费计划 | 零成本，社区集成最好 |
| 小团队（< 10人） | AWS ECR / 自建 Registry | 小规模没必要复杂运维 |
| 中型企业（有合规要求） | Harbor | 内置漏洞扫描 + RBAC + 审计日志 |
| AWS 原生企业 | AWS ECR + Inspector | 深度集成 AWS IAM / VPC / CloudTrail |
| 多云环境 | Harbor (S3 后端) | 统一管理多云的镜像存储 |
| 已有 Nexus/Artifactory | 用现有仓库 | 减少额外运维成本 |

## Dockerfile 编写与实践

### Dockerfile 指令速查

| 指令 | 作用 | 示例 |
|------|------|------|
| `FROM` | 指定基础镜像 | `FROM node:20-alpine` |
| `WORKDIR` | 设置工作目录 | `WORKDIR /app` |
| `COPY` | 复制文件到镜像 | `COPY . .` |
| `RUN` | 在构建时执行命令 | `RUN npm install` |
| `ENV` | 设置环境变量 | `ENV NODE_ENV=production` |
| `EXPOSE` | 声明容器端口 | `EXPOSE 3000` |
| `CMD` | 容器启动时的默认命令 | `CMD ["node", "app.js"]` |
| `ENTRYPOINT` | 容器入口（不可被覆盖） | `ENTRYPOINT ["docker-entrypoint.sh"]` |
| `ARG` | 构建时参数 | `ARG VERSION=1.0.0` |
| `LABEL` | 元数据标签 | `LABEL maintainer="team@example.com"` |
| `HEALTHCHECK` | 健康检查 | `HEALTHCHECK CMD curl -f http://localhost/` |
| `USER` | 指定运行用户 | `USER node` |

### Dockerfile 进化史

**版本 1：初学者**
```dockerfile
FROM node:20
COPY . .
RUN npm install
CMD ["node", "app.js"]
```
问题：镜像巨大、缓存利用差、安全隐患。

**版本 2：进阶**
```dockerfile
FROM node:20-alpine     # 用 Alpine 减小体积
WORKDIR /app
COPY package*.json ./
RUN npm install          # 利用 Docker 分层缓存
COPY . .
EXPOSE 3000
USER node                # 非 root 运行（安全）
CMD ["node", "app.js"]
```

**版本 3：多阶段构建**
```dockerfile
# 第一阶段：构建
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# 第二阶段：运行（极小镜像）
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
USER node
CMD ["node", "app.js"]
```
优势：最终镜像只包含**运行时必需**的文件。

### Dockerfile 反模式清单及修复方案

以下是在大规模容器化实践中总结出的最常见的 Dockerfile 反模式，每个都有对应的修复指引。

| 反模式 | 风险等级 | 症状 | 修复方案 |
|--------|---------|------|---------|
| 使用 `:latest` 标签 | 🔴 严重 | 构建不可复现，今天和明天的镜像不一样 | 锁定到精确版本：`FROM node:20.11.0-alpine3.19` |
| 不设 `.dockerignore` | 🟠 中 | 镜像中打包了 node_modules、.git、日志文件 | 创建 `.dockerignore` 排除无关目录 |
| 每个 RUN 一条指令 | 🟠 中 | 镜像层数过多，但每层没有合理利用缓存 | 合并相关 RUN 指令（`RUN apt-get update && apt-get install -y ...`） |
| COPY . 在安装依赖之前 | 🔴 严重 | 代码变更导致整个依赖层缓存失效 | 先 COPY package.json，安装依赖，再 COPY 源码 |
| 使用 root 用户运行 | 🟠 中 | 容器被攻破后攻击者获得宿主 root 权限 | 添加 USER 指令切换到非 root 用户 |
| 在镜像中存储密钥 | 🔴 严重 | API Key / 数据库密码硬编码在镜像层 | 使用 `docker secret` 或运行时环境变量注入 |
| 基础镜像使用 :full 而非 :slim | 🟠 中 | 包含大量用不到的编译工具和文档 | 选择 Alpine / Slim / Distroless 基础镜像 |
| 不清理包管理器的缓存 | 🟠 中 | `/var/cache/apt` 或 `~/.npm` 残留导致镜像膨大 | `RUN apt-get clean && rm -rf /var/lib/apt/lists/*` |
| ADD 指令代替 COPY | 🟢 低 | ADD 的自动解压和 URL 下载特性容易引发意外 | 除非需要自动解压 tar，否则一直用 COPY |
| RUN chmod 修改权限 | 🟢 低 | 用 RUN 改文件权限导致缓存浪费 | 在 COPY 时用 `--chmod` 参数 |
| 单层超大镜像 | 🔴 严重 | 所有层合并在一个 RUN 指令中，无法共享缓存 | 合理拆分：基础层 → 依赖层 → 应用层 |
| 不使用 LABEL | 🟢 低 | 镜像没有元数据，难以识别维护者、版本、构建日期 | 添加 LABEL 指令记录元信息 |
| 未设置 HEALTHCHECK | 🟠 中 | 容器已死但 Docker 认为 "Up"，启动不检测 | 添加 HEALTHCHECK 指令 |

#### 修复示例：从反模式到最佳实践

**反模式示例**（包含多个问题）：

```dockerfile
FROM node:latest
COPY . /app
RUN npm install
RUN npm run build
CMD node app.js
```

**问题清单**：
1. `node:latest` — 构建不可复现
2. `COPY . /app` — 先于 `RUN npm install`，代码变更导致重新安装
3. 两条分开的 `RUN` — 增加了不必要的层
4. 没有 `WORKDIR` — 工作目录不明确
5. `CMD node app.js` — 未使用 JSON 形式，信号处理有问题
6. 没有 `USER` — root 运行
7. 没有 `.dockerignore` — 会打包 node_modules、.git 等

**修复后**：

```dockerfile
FROM node:20.11.0-alpine3.19 AS builder
WORKDIR /build
COPY package.json package-lock.json ./
RUN npm ci --only=production && npm cache clean --force

FROM node:20.11.0-alpine3.19
WORKDIR /app
COPY --from=builder /build/node_modules ./node_modules
COPY . .
EXPOSE 3000
USER node
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js
CMD ["node", "app.js"]
```

### 镜像构建优化决策树

#### 基础镜像选型决策

```
选择基础镜像
  │
  ├── 需要 glibc 兼容性（大多数 Linux 应用）？
  │   ├── 是 → 继续 ↓
  │   │
  │   ├── 对镜像大小敏感？→ 需要多大？
  │   │   ├── < 10MB 极限优化？
  │   │   │   └── 使用 Distroless（仅运行时二进制）
  │   │   │       └── Go 静态编译：FROM scratch
  │   │   ├── 10-50MB 中等优化？
  │   │   │   └── 使用 Alpine（musl libc，~5MB）
  │   │   │       └── 注意：musl 与 glibc 存在细微 ABI 差异
  │   │   ├── 50-150MB 标准优化？
  │   │   │   └── 使用 Slim 变体（Debian 裁剪版）
  │   │   │       └── 兼容性最好，安全更新及时
  │   │   └── > 150MB 无优化？
  │   │       └── 使用标准镜像（Debian/Ubuntu 完整版）
  │   │
  │   └── 对大小不敏感？
  │       └── 使用标准镜像（兼容性最佳）
  │
  ├── 不需要 glibc（Alpine musl 可以满足）？
  │   └── 首选 Alpine 系列（多阶段构建 + 最终阶段用 Alpine）
  │       └── 注意：Python 有 wheels for manylinux，但 musl 可能需编译
  │
  ├── 需要 GPU / CUDA？
  │   └── 使用 NVIDIA CUDA 官方基础镜像（分层选择）
  │       ├── 极致大小：nvidia/cuda:12.1.0-runtime-ubuntu22.04
  │       └── 含工具链：nvidia/cuda:12.1.0-devel-ubuntu22.04
  │
  └── 静态编译语言（Go / Rust / Zig）？
      └── 最佳方案：FROM scratch（仅包含编译后的二进制文件）
          └── 镜像大小 = 二进制文件大小（通常 < 20MB）
```

#### 基础镜像对比表

| 基础镜像 | 大小（约） | C 库 | 包管理器 | 安全更新 | 兼容性 | 适用场景 |
|---------|-----------|------|---------|---------|-------|---------|
| `scratch` | 0B | 无 | 无 | N/A | 只支持静态编译 | Go/Rust 静态二进制 |
| `alpine:3.19` | ~5MB | musl | apk | 及时 | 一般（musl gotcha） | 追求最小体积 |
| `ubuntu:22.04` | ~77MB | glibc 2.35 | apt | 及时 | 高 | 通用场景 |
| `debian:12-slim` | ~80MB | glibc 2.36 | apt | 及时 | 高 | 标准优化推荐 |
| `debian:12` | ~120MB | glibc 2.36 | apt | 及时 | 最高 | 需要完整工具链 |
| `distroless/base:nonroot` | ~25MB | glibc | 无 | 及时 | 高 | 安全敏感生产环境 |
| `gcr.io/distroless/static-debian12` | ~2MB | 无 | 无 | 及时 | 静态二进制 | 极致的生产安全 |
| `node:20-alpine` | ~120MB | musl | apk | 随 Alpine | Node 应用 | Node.js Alpine |
| `python:3.12-slim` | ~120MB | glibc | apt | 随 Debian | Python 应用 | Python Slim |

> **Note on Alpine**: Alpine 使用 musl libc 而非 glibc，某些 Python wheels（如 `psycopg2-binary`、`cryptography`）在 Alpine 上需要从源码编译，反而会导致构建时间变长。并非所有场景都适合 Alpine。

#### 镜像分层的缓存优化原则

```
分层原则（按变更频率从低到高）：

层 1（最低变率）：基础 OS 层
  → FROM node:20-alpine
  → 几乎不变，缓存命中率 100%

层 2（低变率）：系统依赖
  → RUN apk add --no-cache curl ca-certificates tzdata
  → 只在添加新依赖时变更

层 3（中变率）：应用依赖
  → COPY package.json package-lock.json ./
  → RUN npm ci
  → 只在更新依赖时变更

层 4（高变率）：应用代码
  → COPY . .
  → 每次代码提交都可能变更

层 5（运行时）：配置和入口
  → COPY entrypoint.sh ./
  → CMD ["node", "app.js"]
  → 低变率
```

构建命令优化——利用 BuildKit 的缓存挂载：

```dockerfile
# 启用 BuildKit 缓存（需要 DOCKER_BUILDKIT=1）
# 在多次构建间缓存 apt/npm/pip 包

FROM node:20-alpine AS builder
WORKDIR /app
RUN --mount=type=cache,target=/root/.npm \
    npm ci --cache /root/.npm --prefer-offline
```

## 数据持久化：Volume 与 Bind Mount

容器默认是**无状态**的：重启后数据丢失。

```
┌─────────────────┐
│    Container    │  ← 写入 /app/data
│                 │
├─────────────────┤
│    Volume       │  ← 数据独立于容器生命周期
└─────────────────┘
```

| 方式 | 说明 | 适用场景 |
|------|------|---------|
| Volume（推荐） | Docker 管理的数据卷 | 数据库、持久化存储 |
| Bind Mount | 宿主机目录挂载到容器 | 开发热重载 |
| tmpfs | 内存中的临时存储 | 敏感数据（不写入磁盘） |

```bash
# Volume
docker volume create my-data
docker run -v my-data:/data my-app

# Bind Mount
docker run -v /host/path:/container/path my-app

# tmpfs
docker run --tmpfs /tmp my-app
```

## 网络模式

| 网络模式 | 行为 | 应用场景 |
|---------|------|---------|
| `bridge`（默认） | 通过 Docker 网桥 NAT | 单机容器互联 |
| `host` | 容器直接使用宿主机网络 | 需要高性能网络 |
| `none` | 无网络 | 安全隔离 |
| `overlay` | 跨主机容器网络 | Docker Swarm / K8s |

```bash
# 创建自定义 bridge 网络
docker network create my-network

# 容器加入网络
docker run --network my-network --name web nginx
docker run --network my-network --name api my-api

# 通过服务名通信（Docker DNS）
# web 容器中访问：curl http://api:3000
```

### 容器网络性能对比

不同的网络模式在性能和隔离性上存在显著差异。在选择网络模式时需要权衡吞吐量和延迟。

#### 网络模式基准性能对比

| 网络模式 | TCP 吞吐量（Gbps） | TCP 延迟（μs） | UDP 吞吐量（Mpps） | CPU 开销 | 隔离性 | 适用场景 |
|---------|-------------------|--------------|-------------------|---------|-------|---------|
| **Host** | 39.4（接近线速） | 10-15 | 3.8 | 最低 | 无（与宿主机共享栈） | 高性能网络应用（Nginx/Proxy/LB）、延迟敏感工作负载 |
| **Macvlan（Bridge 模式）** | 38.1 | 12-18 | 3.5 | 低 | 中（独立 L2） | 需要独立 MAC 地址、直接接入物理网络 |
| **Ipvlan（L2 模式）** | 38.5 | 11-16 | 3.6 | 低 | 中（共享 MAC） | Macvlan 替代方案（不需要独立 MAC） |
| **Overlay（VXLAN）** | 28.2 | 35-50 | 2.1 | 中 | 高（封装隔离） | 跨主机容器通信、Docker Swarm |
| **Bridge（默认）** | 34.6 | 20-25 | 3.1 | 低-中 | 中（NAT 隔离） | 默认模式，单机容器互通 |
| **Ipvlan（L3 模式）** | 37.8 | 13-17 | 3.7 | 低 | 中（路由隔离） | L3 分段网络环境 |

> 测试环境：双路 Intel Xeon Gold 6248R (3.0GHz)，64GB RAM，Mellanox ConnectX-6 Dx 25GbE，Linux Kernel 6.2。数值为相对参考，具体值因硬件和内核版本而异。

#### 网络模式选型决策

```
高
│   Host ────── 需要极致性能？容器直接绑定端口
│   │            + 吞吐量最大，延迟最低
│   │            - 无网络隔离，端口冲突风险
│   │
│   Macvlan ─── 需要容器直接暴露在物理网络？
│   │            + 性能接近线速，独立 MAC
│   │            - 需交换机支持，IP 地址消耗大
│   │
│   Ipvlan ──── 大量容器需要独立 IP 但 MAC 地址不够？
│   │            + 共享 MAC，无 MAC 地址耗尽问题
│   │            - 部分网络工具不支持
│   │
│   Overlay ─── 跨主机容器需要互通？
│   │            + 跨主机通信无需端口映射
│   │            - 性能损耗 20-30%，延迟增加
│   │
│   Bridge ──── 默认模式，满足大部分需求？
│   │            + 开箱即用，端口映射灵活
│   │            - 通过 NAT 访问，性能折衷
│   │
低   ─────────────────────────────────────────────
    高          网络隔离性              低
```

#### 生产选型建议

| 场景 | 推荐网络模式 | 配置要点 |
|------|------------|---------|
| Web 服务器（Nginx/HAProxy） | Host 或 Macvlan | Host 模式绑定宿主机 IP，减少 NAT 跳跃 |
| API 微服务（内部通信） | Bridge（自定义） | 创建自定义 bridge 网络，通过服务名 DNS 通信 |
| 数据库容器 | Host 或 Bridge + 持久化 | 尽可能 Host 模式减少网络延迟 |
| 跨主机 Docker Swarm | Overlay | 使用加密 overlay（`--opt encrypted`） |
| 监控/日志采集 | Host | DaemonSet/Service 模式，方便抓取宿主机指标 |
| 开发环境 | Bridge | 端口映射即可，无需额外配置 |

## 容器存储驱动对比

Docker 的存储驱动决定了镜像层和容器可写层的管理方式，直接影响容器的 I/O 性能和磁盘利用率。

### 主流存储驱动概览

| 存储驱动 | Linux 内核要求 | 性能（读/写） | 磁盘利用率 | 层数限制 | 推荐状态 | 适用场景 |
|---------|--------------|-------------|-----------|---------|---------|---------|
| **overlay2** | >= 4.0（推荐 >= 4.9） | ★★★★★ | ★★★★★（共享页缓存） | 无硬限制 | ✅ 推荐（默认） | 所有现代 Linux 发行版 |
| **overlay** | >= 3.18 | ★★★★☆ | ★★★★☆ | 无硬限制 | ❌ 已弃用 | overlay2 不可用时的备选 |
| **fuse-overlayfs** | 任意（用户空间） | ★★★☆☆ | ★★★★☆ | 无限制 | ✅ 推荐（Rootless） | Rootless Docker |
| **devicemapper** | >= 3.8 | ★★☆☆☆ | ★★☆☆☆（loopback 模式） | 受限 | ❌ 强烈不推荐 | 仅遗留系统 |
| **aufs** | >= 3.8（需内核模块） | ★★★★☆ | ★★★★☆ | 42 | ❌ 已弃用 | 淘汰中，不在新内核上支持 |
| **btrfs/zfs** | 对应文件系统 | ★★★☆☆ | ★★★★★（快照） | 无限制 | ⚠️ 特殊场景 | btrfs/zfs 作为底层文件系统的环境 |
| **vfs** | 任意 | ★☆☆☆☆ | ★☆☆☆☆ | 无限制 | ❌ 仅测试 | 调试和测试，无复制写能力 |

### overlay2：行业标准

overlay2 是 Docker 的默认存储驱动，从 Linux 内核 4.0 开始支持（4.9+ 有稳定性和性能改进）。

**工作原理**：
```
overlay2 使用两个目录：
  ├── lowerdir（下层）: 镜像层（只读）
  └── upperdir（上层）: 容器层（可写）

重要特性：
  - 写时复制（Copy-on-Write）：修改文件时复制到上层
  - 共享页缓存（Page Cache）：同文件跨容器共享内存页
  - O_TMPFILE 支持：临时文件不写入底层文件系统
```

**性能优势**：
- 相比 devicemapper，overlay2 的读写性能高出 3-5 倍
- 启动容器时无需预分配空间（devicemapper 需要）
- 页缓存共享：20 个容器运行相同的基础镜像时，共享页缓存节省 90%+ 内存

### devicemapper：为什么强烈不推荐

devicemapper 是 Docker 最早支持的生产存储驱动，但在 loopback 模式下存在严重的性能缺陷：

| 问题 | 说明 | 影响 |
|------|------|------|
| loopback 模式性能极差 | 在稀疏文件上创建文件系统，双重文件系统开销 | I/O 性能下降 50-80% |
| 预分配 100GB | Docker 默认创建 100GB 稀疏文件 | 磁盘空间浪费 |
| 碎片化严重 | 容器大量写入后文件碎片指数级增长 | 写入性能持续劣化 |
| 没有页缓存共享 | 每个容器独立缓存基础镜像 | 内存使用增加数倍 |
| 不支持 O_DIRECT | direct-io 操作需要 direct-lvm 模式 | 复杂配置需求 |

> **如果你还在用 devicemapper（特别是 loopback 模式），强烈建议迁移到 overlay2。** 从 Docker 17.06 起，overlay2 在大部分 Linux 发行版上已经是默认存储驱动。

### 验证当前存储驱动

```bash
docker info --format '{{.Driver}}'
# 输出示例：overlay2

# 查看更详细的存储驱动信息
docker info | grep -A 10 "Storage Driver"
```

### 迁移到 overlay2（如果当前不是）

```bash
# 备份重要数据
# 停止 Docker
systemctl stop docker

# 修改 Docker daemon.json
cat > /etc/docker/daemon.json <<EOF
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

# 清空旧存储目录（注意：会丢失所有本地镜像和容器！）
rm -rf /var/lib/docker

# 重启 Docker
systemctl start docker
```

## Docker Compose：多容器编排

```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/app

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: pass

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

```bash
docker compose up -d     # 启动所有服务
docker compose down      # 停止并清理
docker compose logs -f   # 查看日志
```

## 镜像安全与优化

### 镜像优化

| 优化手段 | 效果 |
|---------|------|
| 选择 Alpine 基础镜像 | 从 ~200MB → ~5MB |
| 多阶段构建 | 移除构建工具和中间文件 |
| 合并 RUN 指令 | 减少镜像层数 |
| .dockerignore | 排除不必要的文件 |
| 使用 --no-cache | 减少安装缓存 |

### 安全扫描

```bash
# Trivy 扫描（推荐的开源工具）
trivy image my-app:latest

# Docker Scout（Docker Desktop 内置）
docker scout quickview my-app:latest
```

**安全基线**：
- 使用官方基础镜像（有漏洞追踪）
- 非 root 运行（`USER` 指令）
- 最小权限文件系统（read-only rootfs）
- 定期扫描 CVE
- 不在镜像中存储敏感信息（密钥、密码）

## 生产案例：某AI公司模型服务的镜像体积从8GB优化到600MB

### 业务背景

**公司概况**：
- 某专注于 NLP 模型推理的 AI 公司（研发 50 人，服务 200+ 企业客户）
- 核心业务：提供基于 GPU 的实时文本分类、情感分析 API
- 技术栈：PyTorch + FastAPI + GPU 推理节点

**迁移前状况**：

| 维度 | 说明 |
|------|------|
| 镜像构建方式 | 每次构建都从 `nvidia/cuda:12.1.0-devel-ubuntu22.04` 开始 |
| 镜像大小 | 8.2GB（未压缩） |
| 包含内容 | CUDA Toolkit 12.1 + cuDNN 8.9 + PyTorch 2.1 + transformers + 模型文件 + 应用代码 |
| 部署方式 | Docker 拉到 GPU 节点，每次 pull 需 5-10 分钟 |
| 部署频率 | 每天 2 次（受限于镜像拉取时间） |
| 迭代模式 | 每次改一行代码也要重新下载 8GB 镜像 |

### 核心痛点

| 痛点 | 严重程度 | 具体影响 |
|------|---------|---------|
| 镜像拉取时间过长 | 🔴 严重 | 每次部署 5-10 分钟卡在 docker pull，GPU 节点带宽打满 |
| 频繁全量下载 | 🔴 严重 | 每周约 500GB 的镜像传输流量，CDN 成本暴涨 |
| 开发迭代效率低 | 🟠 中 | 开发改一行代码 → CI 构建 15 分钟 → 部署 10 分钟 → 验证 5 分钟，一个迭代 30 分钟 |
| 磁盘空间浪费 | 🟠 中 | 每个 GPU 节点保留 3-5 版本镜像，占用 40GB+ 磁盘 |
| 版本回退缓慢 | 🟠 中 | 出问题时切回旧版本也需要重新 pull，难以快速止血 |

### 架构优化方案

#### 分层基础镜像策略

```
优化前（单体镜像 8.2GB）:
┌──────────────────────────────────────────┐
│  模型文件 + 应用代码 (2.5GB)               │
│  Python包 + transformers (1.2GB)          │
│  PyTorch + CUDA扩展 (1.8GB)              │
│  cuDNN 8.9 (0.7GB)                       │
│  CUDA Toolkit 12.1 (2.0GB)               │
└──────────────────────────────────────────┘

优化后（分层镜像，复用基础层）:
┌──────────────────────────────────────────┐
│ 层3: 模型 + 应用代码       → 每次构建变    │
│        大小: 150MB-600MB (按模型)         │
├──────────────────────────────────────────┤
│ 层2: Python依赖 + PyTorch  → 周更          │
│        大小: 1.2GB                        │
├──────────────────────────────────────────┤
│ 层1: CUDA Runtime + cuDNN → 月更          │
│        大小: 1.8GB                        │
│        平台团队维护，预拉取到所有 GPU 节点    │
└──────────────────────────────────────────┘
```

#### 实施细节

**Step 1 — 基础 CUDA 镜像（平台团队维护，月更新）**

```dockerfile
# Dockerfile.cuda-base
# 由平台团队维护，只包含 CUDA Runtime（不含 Devel，节省 ~2GB）
FROM nvidia/cuda:12.1.0-runtime-ubuntu22.04 AS cuda-base

# 安装 cuDNN（仅运行时库，不含开发头文件）
RUN apt-get update && apt-get install -y --no-install-recommends \
    libcudnn8=8.9.7.29-1+cuda12.1 \
    && rm -rf /var/lib/apt/lists/*

# 系统优化
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl tzdata \
    && rm -rf /var/lib/apt/lists/*

LABEL maintainer="platform-team@company.com"
LABEL cuda-version="12.1.0"
LABEL cudnn-version="8.9.7.29"
```

构建发布管道：

```bash
# 每月构建一次
docker build -f Dockerfile.cuda-base -t registry.company.com/cuda-base:12.1.0-runtime-20240301 .
docker push registry.company.com/cuda-base:12.1.0-runtime-20240301

# 预拉取到所有 GPU 节点
for node in gpu-node-1 gpu-node-2 gpu-node-3; do
  ssh $node "docker pull registry.company.com/cuda-base:12.1.0-runtime-20240301"
done
```

**Step 2 — Python 依赖层（利用 Docker 分层缓存）**

```dockerfile
# Dockerfile.deps
FROM registry.company.com/cuda-base:12.1.0-runtime-20240301 AS deps

WORKDIR /app

# 使用 conda-pack 压缩 Python 环境
COPY environment.yml conda-lock.yml ./

RUN conda env create -f environment.yml && \
    conda-pack -n inference-env -o /tmp/inference-env.tar.gz && \
    conda clean --all -y

# 构建为可复用的依赖层镜像
```

使用 conda-pack 的关键优势：

```bash
# conda-pack 打包的 Python 环境可以解压即用
# 相比 conda install，打包后体积减少 40%
# 且解压无需联网，部署更快

# 构建依赖层镜像
docker build -f Dockerfile.deps \
  -t registry.company.com/inference-deps:20240301 \
  .

# 依赖层仅在 Python 包变更时重新构建
# 通常每周 1-2 次
```

**Step 3 — 模型和应用层（动态加载 + 多阶段构建）**

```dockerfile
# Dockerfile.app
# 构建阶段
FROM registry.company.com/inference-deps:20240301 AS builder

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# 运行阶段
FROM registry.company.com/inference-deps:20240301

WORKDIR /app
COPY --from=builder /opt/conda/envs/inference-env /opt/conda/envs/inference-env

# 应用代码（高变率层）
COPY app/ ./app/
COPY config/ ./config/

# 模型文件通过独立 Volume 或启动时下载（动态加载）
# 不在镜像中包含模型文件！！！
ENV MODEL_PATH=/models/current
ENV CUDA_VISIBLE_DEVICES=0

USER 1001:1001

HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

ENTRYPOINT ["python", "-m", "app.main"]
```

**Step 4 — 模型动态加载（关键优化）**

```
模型文件的处理策略：

方案 A（之前在镜像中打包模型 — 不推荐）：
  每次模型更新 → 重新构建镜像 → 所有节点重新下载 → 2.5GB 模型文件
  ❌ 构建时间 +15 分钟，拉取时间 +3 分钟

方案 B（动态加载 — 推荐方案）：
  镜像只包含应用代码
  模型存储在共享文件系统（NFS / S3 / EFS）
  容器启动时或按需从共享存储加载模型
  ✅ 模型更新无需重建镜像，秒级切换
```

动态加载实现：

```python
# app/model_loader.py
import os
import hashlib
import boto3
from pathlib import Path

class ModelLoader:
    """按需加载模型，支持版本切换"""

    def __init__(self, model_registry_url: str, cache_dir: str = "/models/cache"):
        self.s3 = boto3.client("s3")
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(parents=True, exist_ok=True)
        self.current_model = None

    def load_model(self, model_version: str):
        """从 S3 加载指定版本的模型"""

        model_path = self.cache_dir / f"model-{model_version}"

        if model_path.exists():
            # 缓存命中
            self.current_model = self._load_from_path(model_path)
            return self.current_model

        # 从 S3 下载
        self.s3.download_file(
            Bucket="company-models",
            Key=f"models/{model_version}/model.pt",
            Filename=str(model_path)
        )

        self.current_model = self._load_from_path(model_path)
        return self.current_model
```

### 关键指标前后对比

| 指标 | 优化前 | 优化后 | 改善幅度 |
|------|--------|--------|---------|
| 镜像大小（未压缩） | 8.2 GB | 600 MB | 92.7% |
| 镜像拉取时间 | 5-10 分钟 | 18 秒（仅差异层） | 97% |
| 每日部署次数 | 2 次 | 20 次 | 10 倍 |
| 构建时间 | 15 分钟 | 4 分钟 | 73% |
| 迭代周期（改代码到验证） | 30 分钟 | 8 分钟 | 73% |
| 每节点磁盘占用（保留 5 版本） | 41 GB | 5 GB（分层共享） | 88% |
| 月度镜像传输量 | ~500 GB | ~30 GB（仅应用层变化） | 94% |
| 模型版本切换速度 | 30 分钟（重建镜像+部署） | 15 秒（动态加载新版本） | 99% |

### 经验教训

1. **"胖基础镜像"策略是反模式**
   - 之前试图维护一个"无所不包"的基础镜像，最终变成了每周都有废弃代码的胖镜像
   - 教训：基础镜像应该只包含**操作系统层 + CUDA Runtime**，其余分层管理

2. **模型文件不应该进镜像**
   - 这是最大的优化点。模型文件占 2.5GB，而且频繁更新
   - 教训：将模型文件视为**数据**而非**代码**，通过共享存储动态加载

3. **CUDA Devel 镜像大部分用不到**
   - 生产环境只需要 CUDA Runtime（~1.8GB），不需要 CUDA Toolkit（~4GB）
   - 教训：`nvidia/cuda:12.1.0-runtime-ubuntu22.04` 而非 `-devel-` 版本

4. **conda-pack 是 Python 场景的利器**
   - 将 conda 环境打包为压缩包，解压即用
   - 相比 `pip install` + `conda install`，conda-pack 的压缩率更好（无缓存文件）
   - 教训：conda-pack 适合频繁部署的 Python 应用场景

5. **分层缓存需要 CI/CD 配合**
   - 基础层在 CI 中单独构建、单独推送
   - 依赖层使用 `--cache-from` 参数利用 BuildKit 远程缓存
   - 教训：构建策略与部署策略同等重要

6. **镜像拉取并行化**
   - 在 GPU 节点上预拉取基础层（cuda-base + deps），应用层部署时只需拉取 ~150MB
   - 使用 `docker pull` 的并发下载特性（Docker 20.10+ 默认开启）
   - 教训：预拉取计划可以消除 90% 的拉取等待时间

## 本章小结

| 要点 | 说明 |
|------|------|
| 容器本质 | 隔离的进程，而非轻量虚拟机 |
| 镜像分层 | 层叠只读文件系统，可共享和缓存 |
| Dockerfile | 多阶段构建、Alpine 基础镜像、分层缓存 |
| 反模式清单 | 避免 :latest、先装依赖再拷代码、非 root 运行 |
| 存储驱动 | overlay2 是行业标准，取代 deprecated 的 aufs 和 devicemapper |
| 网络选型 | host 要性能、bridge 默认安全、overlay 跨主机 |
| 镜像仓库 | Harbor 适合企业合规，ECR 适合 AWS 原生，小团队自建即可 |
| 生命周期 | create → start → stop → rm |
| 数据持久化 | Volume（推荐）、Bind Mount、tmpfs |
| 网络 | bridge（默认）、host、overlay |
| Compose | YAML 描述多服务应用 |
| 安全 | 非 root、最小镜像、定期扫描 |
| 镜像优化实例 | 分层基础镜像 + 动态模型加载，8GB → 600MB |

