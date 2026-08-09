# 第7章：容器编排与 Kubernetes

## 本章学习目标

- 理解容器编排解决的核心问题
- 掌握 Kubernetes 核心架构与组件职责
- 了解 Pod、Service、Deployment 等核心资源
- 理解 ConfigMap、Secret 的配置管理方式

## 为什么需要容器编排

### 单机 Docker 的局限性

| 问题 | 描述 |
|------|------|
| 宿主机宕机 | 容器全部不可用，无自动迁移 |
| 扩缩容困难 | 需要手动启动/停止容器 |
| 服务发现 | 容器 IP 变动频繁，难以维护 |
| 负载均衡 | 需要额外工具（Nginx / HAProxy）|
| 滚动更新 | 手动编排麻烦 |
| 存储编排 | 多节点共享数据困难 |
| 配置管理 | 几百个容器的配置如何统一管理 |

**容器编排平台（K8s）就是要解决这些问题。**

## Kubernetes 核心架构

```
┌─────────────────────────────────────────┐
│            Control Plane                 │
│  ┌──────┐  ┌──────┐  ┌───────────┐     │
│  │etcd │  │APIServer│  │Scheduler  │     │
│  └──────┘  └──────┘  └───────────┘     │
│  ┌──────┐  ┌──────┐                     │
│  │CM   │  │Controller Mgr│              │
│  └──────┘  └─────────────┘             │
└──────────────┬─────────────────────────┘
               │ API（HTTPS）
               ▼
┌─────────────────────────────────────────┐
│            Worker Node                   │
│  ┌─────────────┐  ┌─────────────┐       │
│  │  kubelet    │  │  kube-proxy │       │
│  └─────────────┘  └─────────────┘       │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐           │
│  │Pod │ │Pod │ │Pod │ │Pod │           │
│  │    │ │    │ │    │ │    │           │
│  └────┘ └────┘ └────┘ └────┘           │
│  ┌────────────────────────────────┐     │
│  │   Container Runtime (containerd)│     │
│  └────────────────────────────────┘     │
└─────────────────────────────────────────┘
```

### Control Plane（控制平面）

| 组件 | 职责 | 一句话理解 |
|------|------|-----------|
| **API Server** | 所有操作的入口 | K8s 的"HTTP 服务器" |
| **etcd** | 集群状态存储 | K8s 的"数据库" |
| **Scheduler** | 决定 Pod 调度到哪个节点 | K8s 的"调度员" |
| **Controller Manager** | 维护集群期望状态 | K8s 的"监控循环" |
| **Cloud Controller Manager** | 与云平台 API 交互 | 云厂商集成层 |

### Worker Node（工作节点）

| 组件 | 职责 |
|------|------|
| **kubelet** | 节点上的"代理"，确保 Pod 按预期运行 |
| **kube-proxy** | 维护网络规则，实现 Service 的负载均衡 |
| **Container Runtime** | 容器引擎（containerd / CRI-O） |

## 核心资源对象

### Pod（最小调度单元）

Pod 是 K8s 中**最小、最基本的调度单元**，包含 **一个或多个容器**（共享网络栈和存储卷）。

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: web
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

> **重要**：通常不直接创建 Pod，而是通过更高级的资源（Deployment）管理。

### Deployment（无状态工作负载）

**Deployment** 是管理 Pod 副本和滚动更新的核心资源。

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3          # 期望的 Pod 数量
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 滚动升级中最多可超出期望副本数
      maxUnavailable: 0  # 滚动升级中最多不可用数量
```

**Deployment 的能力**：
- 声明式滚动更新
- 自动回滚（保留历史版本）
- 水平扩缩容
- 自愈（Pod 挂了重新创建）

### Service（服务发现与负载均衡）

Pod 的 IP 是随机的、会变化的。**Service** 提供稳定的网络入口。

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80          # Service 端口
    targetPort: 80    # Pod 容器端口
  type: ClusterIP     # Service 类型
```

**Service 类型**：

| 类型 | 访问方式 | 使用场景 |
|------|---------|---------|
| **ClusterIP**（默认） | 集群内部 VIP | 内部服务通信 |
| **NodePort** | 节点 IP + 端口 | 外部调试/测试 |
| **LoadBalancer** | 云厂商负载均衡器 | 对外暴露服务 |
| **ExternalName** | DNS CNAME | 访问外部服务 |

### Ingress（HTTP 路由）

Ingress 将外部 HTTP/HTTPS 请求路由到集群内部 Service。

```
用户 → 域名 → Ingress Controller → 路由规则 → Service → Pod
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

常见的 Ingress Controller：**Nginx Ingress**、**Traefik**、**Istio Gateway**

### ConfigMap 与 Secret（配置管理）

**ConfigMap** 存储非敏感配置，**Secret** 存储敏感信息。

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    log.level=INFO
    max.connections=100
```

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64 编码
```

**注入方式**：
```yaml
# 环境变量注入
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password

# 文件挂载
volumeMounts:
  - name: config
    mountPath: /etc/config
```

## K8s 发行版选型对比

Kubernetes 生态中有多种发行版可供选择，从轻量级的单机方案到企业级的生产方案，选择取决于团队规模、运维能力和业务需求。

### 主流发行版全景

| 发行版 | 类型 | 安装复杂度 | 控制平面规模 | 生产就绪度 | 社区活跃度 |
|-------|------|-----------|------------|-----------|----------|
| **kubeadm** | 社区标准 | 中（手动 3 节点） | 高可用单集群 | ★★★★☆ | ★★★★★ |
| **K3s** | 轻量级（Rancher） | 极低（单命令） | 单节点/嵌入式 etcd | ★★★★☆ | ★★★★★ |
| **RKE2** | 企业级（Rancher） | 中（自动部署） | 高可用 | ★★★★★ | ★★★★☆ |
| **EKS** | 托管（AWS） | 低（CLI/Console） | AWS 管理 | ★★★★★ | ★★★★★ |
| **AKS** | 托管（Azure） | 低（CLI/Console） | Azure 管理 | ★★★★★ | ★★★★★ |
| **GKE** | 托管（GCP） | 低（CLI/Console） | Google 管理 | ★★★★★ | ★★★★★ |
| **OpenShift** | 企业平台（Red Hat） | 高（全功能平台） | 高可用 | ★★★★★ | ★★★★☆ |
| **MicroK8s** | 轻量级（Canonical） | 极低（snap） | 单节点 | ★★★☆☆ | ★★★☆☆ |
| **k0s** | 轻量级（Mirantis） | 低（单二进制） | 单节点/多节点 | ★★★★☆ | ★★★☆☆ |

### 关键维度详细对比

#### 安装与部署复杂度

| 发行版 | 初始化步骤 | 时间成本 | 需要网络 | 离线安装 | 自动化能力 |
|-------|-----------|---------|---------|---------|----------|
| K3s | 1 条命令（服务器 + Agent） | 5 分钟 | 需要 | 支持（air-gap 模式） | Ansible 剧本广泛 |
| kubeadm | 5-8 步（容器运行时 + 初始化 + CNI） | 30-60 分钟 | 需要 | 支持（镜像预拉取） | kubeadm + Ansible |
| RKE2 | 配置文件 + systemd 服务 | 15-30 分钟 | 部分（可离线部署包） | 原生支持 | Rancher Cluster API |
| EKS | 3 步（eksctl/Console + nodegroup） | 15-20 分钟 | 需要 | 不支持 | eksctl / Terraform |
| GKE | 2 步（gcloud + 节点池） | 5-10 分钟 | 需要 | 不支持 | gcloud / Terraform |
| OpenShift | 20+ 步（安装前检查 + 配置文件 + 部署） | 3-8 小时 | 部分 | 原生支持 | OpenShift Installer / Ansible |

#### 升级策略对比

| 发行版 | 升级方式 | 回滚能力 | 升级控制面 | 升级节点 | 兼容性跟踪 |
|-------|---------|---------|----------|---------|----------|
| kubeadm | `kubeadm upgrade apply` | 不支持（需备份恢复） | 手动 | 手动 drain + upgrade | 社区测试矩阵 |
| K3s | `curl -sfL ... | sh -s - --version` | 支持（重新安装旧版本） | 自动 | 自动（server-agent 协议） | Rancher 测试 |
| RKE2 | 配置文件版本 + 节点滚动 | 支持（修改配置回退） | 滚动 | 自动 | Rancher QA |
| EKS | AWS Console / CLI / IaC | 支持（版本冻结 + 节点组回退） | AWS 管理（滚动） | 自动（节点组替换） | AWS 长期支持 |
| GKE | `gcloud container clusters upgrade` | 支持（快速回退） | Google 管理（自动） | 自动（节点池替换） | Google 长期支持 |
| OpenShift | OTA（Over-the-Air）Web Console | 支持（Admin Complete 回退） | 自动 | 自动（MachineConfigPool） | Red Hat 测试 |

#### 安全性基线

| 发行版 | 默认 Pod 安全 | 控制面加固 | etcd 加密 | 审计日志 | 镜像签名 |
|-------|-------------|-----------|----------|---------|---------|
| K3s | Pod Security Standards（内置） | 内嵌 kube-bench 防御 | 支持 | 支持 | 支持（containerd） |
| RKE2 | 强制 Pod Security Admission | CIS Benchmark 默认 | 强制 | 支持 | 支持 |
| EKS | Pod Security Groups（AWS 安全组） | AWS 管理 | 支持（KMS 加密） | CloudTrail 审计 | ECR 签名 + OPA |
| GKE | GKE Sandbox / Workload Identity | Google 管理 | 默认加密 | Cloud Audit Logs | Binary Authorization |
| OpenShift | Security Context Constraints（SCC） | 默认 SELinux | 支持 | 内置审计 | Red Hat 签名 + 自定义 |

#### 成本对比（月度）

| 发行版 | 控制平面费用（3 节点） | 工作节点费用 | 附加管理费用 | 维护人力（人·天/月） | 隐藏成本 |
|-------|---------------------|------------|------------|-----------------|---------|
| K3s（自建） | 3 台服务器成本 | 按需 | 0 | 2-4 | etcd 备份、监控 |
| kubeadm（自建） | 3 台服务器成本 | 按需 | 0 | 3-5 | 升级测试、兼容性 |
| RKE2（自建） | 3 台服务器成本 | 按需 | 0 | 2-3 | Rancher 高级功能付费 |
| EKS | $0.10/小时/集群（~$73/月） | 按需 | 0 | 0.5-1 | DataTransfer 出站费 |
| AKS | 0（控制面免费） | 按需 | 0 | 0.5-1 | Azure 资源费用 |
| GKE | $0.10/小时/集群（~$73/月） | 按需 | 0 | 0.5-1 | 网络出站费 |
| OpenShift | 订阅 $15,000+/年（3 节点） | 按需 | $5,000+ | 0.5-1 | 中间件授权 |

### 选型建议

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 个人学习/开发 | K3s 或 MicroK8s | 安装简单，资源占用低 |
| 边缘计算 / IoT | K3s | 极轻量级，支持 ARM 架构 |
| 中小团队自建生产 | RKE2 或 kubeadm + Ansible | 生产就绪度高，控制权在自己 |
| AWS 原生企业 | EKS | 深度集成 IAM / VPC / ALB，运维量最低 |
| Azure 原生企业 | AKS | 深度集成 Azure AD / Managed Identity |
| GCP 原生企业 | GKE | Autopilot 模式几乎零运维 |
| 金融/政府合规要求 | OpenShift 或 RKE2 + Rancher | 安全基线最完善，SCC 策略精细 |
| 多云/混合云 | RKE2 + Rancher | 统一管理多个集群 |

## Pod 资源管理决策框架

### Requests 和 Limits 的核心概念

Kubernetes 通过 **Requests**（请求量）和 **Limits**（上限）来控制每个 Pod 的资源使用：

| 概念 | CPU | 内存 |
|------|-----|------|
| **Requests** | 调度依据—调度器保证节点至少有此空闲资源 | 调度依据 + 内存 Guarantee |
| **Limits** | CPU 可以 Burst 到 Limits（如果节点有空闲 CPU），但受 CFS Quota 约束 | 内存达到 Limits 后触发 OOM Kill |

### Resources 设置原则

#### 原则 1：避免"零 Request"

```yaml
# ❌ 错误：不设置 requests/limits
resources: {}
# → 可能被调度到任何节点，但节点压力大时被 Evict

# ✅ 正确：至少设置 requests
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
```

#### 原则 2：CPU 的 Requests = Limits 避免 Throttling

```
CPU 限制行为：

Requests=100m, Limits=200m:
  - 空闲节点：Pod 可以使用 ~200m（Burst）
  - 节点压力大：Pod 被限制在 ~100m
  - 问题：间歇性 Burst → 突发的 CFS 配额限制 → CPU Throttling

Requests=200m, Limits=200m:
  - Pod 始终获得稳定的 200m CPU
  - 没有 Burst 带来的 Throttling 风险
  - 适合延迟敏感型服务
```

**CPU Throttling 诊断**：

```bash
# 容器内查看 CPU Throttling
cat /sys/fs/cgroup/cpu.stat | grep nr_throttled

# PromQL 查询 K8s 级别的 Throttling（需配置 cadvisor metrics）
rate(container_cpu_cfs_throttled_seconds_total[5m]) / on (pod) rate(container_cpu_cfs_periods_total[5m])

# 如果 throttled 占比超过 5%，说明 CPU Limits 设置过紧
```

**设置建议**：

| 服务类型 | CPU Requests : Limits | 理由 |
|---------|---------------------|------|
| 延迟敏感（API Gateway、实时推理） | 1:1（相等） | 确保 CPU 稳定，避免 Throttling 引发 P99 延迟抖动 |
| 批处理（数据管道、定时任务） | 1:2 或 1:4 | 可以 Burst，延迟敏感度低 |
| 后台 Worker（消息消费） | 1:2 ~ 1:3 | 合理利用节点空闲 CPU |
| Sidecar（日志采集、监控） | Requests 适中，Limits 放宽 | 通常轻量，偶尔需要 CPU 处理突发日志 |

#### 原则 3：内存的 Requests = Limits（反对 Burst）

```yaml
# ❌ 错误：内存 Burst 的风险
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
# → 应用使用超过 256Mi 后，节点内存压力触发 OOM Kill

# ✅ 正确：内存 Requests = Limits
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "512Mi"
# → 应用最多使用 512Mi，超限后 OOM Kill（可控），OOM Score 优先级较低
```

**内存 Burst 的真实风险**：

```
节点可用内存 1GB
Pod A: Requests=256Mi, Limits=512Mi (当前使用 480Mi)
Pod B: Requests=256Mi, Limits=512Mi (当前使用 200Mi)

突然 Pod B 内存飙升到 500Mi
→ 节点已使用: 480+500 = 980Mi (接近上限)
→ 如果其他 Pod 再申请内存 → 节点 OOM
→ 内核 OOM Killer 选择杀掉一个 Pod（不一定按你的期望）
```

**OOM Kill 预防检查**：

```bash
# 查看被 OOM 杀掉的 Pod
kubectl get events --all-namespaces --field-selector reason=OOMKilling

# 节点内存压力事件
kubectl get events --all-namespaces --field-selector reason=SystemOOM
```

### 资源管理决策框架

```
Pod 资源设置流程：
     │
     ▼
┌────────────────────────────────────────┐
│ 1. 确定服务类型                          │
│    ├─ 延迟敏感型（API、实时推理）         │
│    ├─ 批处理（数据管道、ETL）            │
│    ├─ 后台 Worker（消息消费）            │
│    └─ Sidecar（日志、监控）              │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 2. 分析历史用量（至少 7 天数据）          │
│    ├─ 取 P95/P99 作为 Requests 基准      │
│    ├─ 取 P99.9 作为 Limits 上限          │
│    └─ 使用 VPA 推荐值作为参考             │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 3. 应用设置规则                          │
│    ├─ CPU: 延迟敏感型 1:1，其他可放宽     │
│    ├─ 内存: 始终 1:1（防止 OOM 误杀）    │
│    └─ Requests > 0（确保调度）           │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 4. 监控调整（持续观察）                   │
│    ├─ CPU Throttling > 5% → 调大 CPU    │
│    ├─ Memory 使用 > 80% Requests → 调大  │
│    └─ Memory 使用 < 40% Requests → 调小  │
└────────────────────────────────────────┘
```

### 常见问题及解决方案

| 症状 | 原因 | 修复 |
|------|------|------|
| Pod 频繁 Pending | Requests 总和 > 节点容量 | 调整 Requests 或扩节点 |
| CPU Throttling 高 | CPU Limits = Requests 但太小 | 增加 CPU Requests |
| 偶发高延迟 | CPU Burst 导致 CFS 配额限制 | 改为 1:1 CPU 设置 |
| Pod OOM 被杀 | 内存使用超过 Limits | 分析内存 Profile 后调大 Limits |
| 节点资源利用率低 | Requests 设置过大 | 使用 VPA 分析减少 Requests |
| Pod 被 Evict | 节点内存压力 | 检查节点内存、配置 QoS Guaranteed |
| 频繁重启（CrashLoopBackOff） | OOM Kill 或存活探测失败 | 查看 `kubectl logs --previous` |

## 工作负载资源对比

| 资源类型 | 适用场景 | 无状态/有状态 |
|---------|---------|-------------|
| **Deployment** | 无状态应用（Web Server、API） | 无状态 |
| **StatefulSet** | 有状态应用（数据库、消息队列） | 有状态 |
| **DaemonSet** | 每节点运行一个（日志采集、监控 Agent） | 无状态 |
| **Job** | 一次性任务（数据迁移、批处理） | 无状态 |
| **CronJob** | 定时任务（定期备份、清理） | 无状态 |

## 存储：PV 与 PVC

```
Pod ──→ PVC (声明需求) ──→ PV (实际存储)
                            ├── NFS
                            ├── Ceph
                            ├── AWS EBS
                            └── 本地存储
```

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```yaml
# pod 挂载 PVC
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

## Namespace：多租户隔离

Namespace 实现集群内的逻辑隔离：

```bash
kubectl get ns          # 查看所有 Namespace
kubectl get pods -n dev # 查看 dev 命名空间的 Pod
```

**常见 Namespace 划分**：

| Namespace | 用途 |
|-----------|------|
| `default` | 默认，无特殊归属 |
| `kube-system` | K8s 系统组件 |
| `kube-public` | 公开资源（所有用户可读）|
| `dev / staging / prod` | 环境隔离 |
| `team-a / team-b` | 团队隔离 |

## K8s 网络方案对比

Kubernetes 本身不实现 Pod 网络，而是通过 **CNI（Container Network Interface）** 插件来提供网络功能。选择 CNI 插件是集群网络设计最关键的决策之一。

### 主流 CNI 插件对比

| 对比维度 | Calico | Flannel | Cilium | Weave Net | Canal (Flannel + Calico) |
|---------|--------|---------|--------|----------|------------------------|
| 核心原理 | BGP 路由 + iptables/eBPF | VXLAN/Host-GW 隧道 | eBPF 内核编程 | VXLAN + weave router | Flannel 网络 + Calico Policy |
| 性能 | ★★★★★（BGP 直连适合大规模） | ★★★☆☆（隧道封装开销） | ★★★★★★（eBPF 线速） | ★★★★☆（快速路径） | ★★★☆☆ |
| 功能丰富度 | ★★★★★（NetworkPolicy + 安全 + 多网络） | ★★☆☆☆（仅基本网络） | ★★★★★★（安全 + 可观测 + 服务网格） | ★★★★☆（加密 + DNS） | ★★★★☆ |
| 可观测性 | ★★★☆☆（Felix 指标） | ★☆☆☆☆（基本） | ★★★★★★（Hubble：完整网络可视化） | ★★★☆☆ | ★★☆☆☆ |
| 安全策略 | ★★★★★（K8s NetworkPolicy + Calico 自定义策略） | ★☆☆☆☆（无 NetworkPolicy） | ★★★★★★（L3-L7 策略 + DNS 策略） | ★★★★☆（加密通信） | ★★★★☆（Calico Policy） |
| 安装复杂度 | 中（需配置 BGP、IP Pool） | 极低（一条命令） | 中-高（需内核 eBPF 支持） | 低 | 低 |
| 大规模支持 | ★★★★★（10,000+ 节点已验证） | ★★★☆☆（1,000+ 节点） | ★★★★★（eBPF 扩展性好） | ★★★★☆ | ★★★☆☆ |
| eBPF 要求 | 可选（eBPF 模式需内核 5.3+） | 不依赖 | 强制（内核 5.10+ 推荐） | 不依赖 | 不依赖 |
| 加密选项 | WireGuard 集成 | 无 | IPsec/WireGuard eBPF | 内置加密 | 无 |
| 多网卡支持 | 支持 | 不支持 | 支持 | 支持 | 有限 |

### 性能基准对比

```
CNI 插件吞吐量 & 延迟对比（裸金属 100GbE）：

Calico (BGP):         ────────────────── 38.2 Gbps (P99 Latency: 35μs)
Calico (eBPF):        ──────────────────── 41.5 Gbps (P99 Latency: 28μs)
Cilium (eBPF):        ────────────────────── 42.8 Gbps (P99 Latency: 25μs)
Flannel (VXLAN):      ──────────────── 31.6 Gbps (P99 Latency: 52μs)
Flannel (Host-GW):    ──────────────────── 40.1 Gbps (P99 Latency: 30μs)
Weave Net:            ────────────── 28.9 Gbps (P99 Latency: 58μs)
Canal:                ──────────────── 30.2 Gbps (P99 Latency: 55μs)

数据传输大小：1024 字节 TCP 流，Pod-to-Pod 跨节点
```

### 网络方案场景匹配

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 小型集群（< 50 节点） | Flannel | 最简单，安装一条命令，足够满足需求 |
| 中型集群（50-500 节点） | Calico（BGP 模式） | 性能好，支持 NetworkPolicy，不需要 eBPF 内核要求 |
| 大型集群（500+ 节点） | Calico（BGP）或 Cilium | BGP 路由在大规模下性能出色，Cilium eBPF 更优 |
| 安全敏感（金融/合规） | Cilium 或 Calico（eBPF） | L3-L7 策略 + DNS 策略 + 透明加密 |
| 需要深度可观测性 | Cilium + Hubble | 服务拓扑图、流量可视化、安全事件回溯 |
| 已有 Istio/Envoy | Cilium | Cilium 与 Istio 集成最佳，可以替换 sidecar |
| 低版本内核（< 4.18） | Flannel 或 Calico（iptables 模式） | eBPF 需要 5.3+ 内核 |
| 边缘/ARM 节点 | Flannel 或 Calico | Cilium 对 ARM 支持也在完善，但 Flannel 最兼容 |
| 多集群/混合云 | Calico | 跨集群网络策略统一管理，Project Calico 支持多集群 |
| GPU 集群 | Calico（BGP 直连） | 减少 VXLAN 隧道开销，GPU 通信延迟敏感 |

### 安装对比

```bash
# Flannel（最简单）
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Calico（标准）
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27/manifests/tigera-operator.yaml
# 然后配置自定义资源

# Cilium（推荐方式）
cilium install --version 1.15.0 \
  --set ipam.mode=kubernetes \
  --set k8sServiceHost=192.168.1.100 \
  --set k8sServicePort=6443
```

## 生产集群高可用设计

### 控制平面高可用架构

#### 多控制平面节点配置

```
生产集群控制平面最小配置：

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Control-1   │  │ Control-2   │  │ Control-3   │
│ us-east-1a  │  │ us-east-1b  │  │ us-east-1c  │
├─────────────┤  ├─────────────┤  ├─────────────┤
│ etcd        │─►│ etcd        │──│ etcd        │
│ API Server  │──│ API Server  │──│ API Server  │
│ Scheduler   │──│ Scheduler   │──│ Scheduler   │
│ CM          │──│ CM          │──│ CM          │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
              ┌─────────┴─────────┐
              │  Load Balancer     │
              │  (AWS NLB /        │
              │   HAProxy /        │
              │   keepalived)      │
              └─────────┬─────────┘
                        │ API:6443
                        ▼
              ┌─────────────────────┐
              │   Worker Nodes       │
              │   (3+ nodes)         │
              └─────────────────────┘
```

**配置要求**：

| 组件 | 最小节点数 | 推荐节点数 | 可用区分布 | 影响说明 |
|------|----------|----------|-----------|---------|
| etcd | 3 | 3 或 5 | ≥ 3 个 AZ | 3 节点允许 1 个故障；5 节点允许 2 个故障 |
| API Server | 2 | 3 | ≥ 2 个 AZ | 配合负载均衡器，任意一个宕机不影响 |
| Scheduler | 2 | 3 | ≥ 2 个 AZ | 选主模式，只有 Leader 工作 |
| Controller Manager | 2 | 3 | ≥ 2 个 AZ | 选主模式，只有 Leader 工作 |

### etcd 备份与恢复

etcd 是 K8s 集群的"大脑"，备份 etcd 是集群高可用的生命线。

#### 备份策略

```bash
# 定期快照（推荐：每 30 分钟到 1 小时）
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd/snapshot-$(date +%Y%m%d-%H%M%S).db

# 备份保留策略
# 保留最近 48 小时的每小时快照
# 保留最近 30 天的每日快照
# 至少一份异地备份（不同区域/云）

# 自动备份脚本（CronJob 定时执行）
cat > /usr/local/bin/etcd-backup.sh <<'EOF'
#!/bin/bash
BACKUP_DIR="/backup/etcd"
DATE=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_FILE="${BACKUP_DIR}/snapshot-${DATE}.db"

# 创建快照
ETCDCTL_API=3 etcdctl snapshot save $SNAPSHOT_FILE \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 验证快照
ETCDCTL_API=3 etcdctl snapshot status $SNAPSHOT_FILE

# 上传到 S3
aws s3 cp $SNAPSHOT_FILE s3://company-etcd-backup/

# 删除本地 7 天前的备份
find $BACKUP_DIR -name "snapshot-*.db" -mtime +7 -delete
EOF

# 设置定时任务（每 30 分钟）
echo "*/30 * * * * root /usr/local/bin/etcd-backup.sh" >> /etc/crontab
```

#### etcd 恢复

```bash
# 场景：etcd 数据损坏或集群不可用

# 停止所有 Control Plane 组件
systemctl stop kube-apiserver kube-scheduler kube-controller-manager
systemctl stop etcd  # 或 docker stop etcd

# 从备份快照恢复 etcd
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd/snapshot-20240101-120000.db \
  --data-dir=/var/lib/etcd-restored \
  --name=control-1 \
  --initial-cluster=control-1=https://192.168.1.10:2380,control-2=https://192.168.1.11:2380,control-3=https://192.168.1.12:2380 \
  --initial-cluster-token=etcd-cluster \
  --initial-advertise-peer-urls=https://192.168.1.10:2380

# 将恢复的数据移到 etcd 数据目录
mv /var/lib/etcd /var/lib/etcd.bak
mv /var/lib/etcd-restored /var/lib/etcd

# 在控制平面的其他节点重复 restore 操作
# 重启所有 Control Plane 组件

# 重要：恢复后需要检查核对该时间点后的资源变更
kubectl get all --all-namespaces
```

#### etcd 健康指标

```bash
# 检查 etcd 集群健康
ETCDCTL_API=3 etcdctl endpoint health --cluster

# 检查 leader 和 follower 状态
ETCDCTL_API=3 etcdctl endpoint status --cluster --write-out=table
# +---------------------+--------+---------+--------+------------+-----------+------------+
# |     ENDPOINT        |  GRPC  |  RAFT   | TERMS  |  STORAGE   |   DB SIZE |  CLUSTER   |
# +---------------------+--------+---------+--------+------------+-----------+------------+
# | 192.168.1.10:2379   | true   |  true   | 42     |  1.2 GB    |  256 MB   |  leader    |
# | 192.168.1.11:2379   | true   |  true   | 42     |  1.2 GB    |  256 MB   |  follower  |
# | 192.168.1.12:2379   | true   |  true   | 42     |  1.2 GB    |  256 MB   |  follower  |
# +---------------------+--------+---------+--------+------------+-----------+------------+

# 推荐的告警阈值
# - DB SIZE > 2GB → 需要 defrag（碎片整理）
# - RAFT term 超过 10 秒未 Commit → 集群性能问题
# - Follower 比 Leader 的 DB SIZE 差异 > 20% → 检查网络/IO
```

### 灾难恢复（DR）策略

#### 多集群容灾模型

| DR 模型 | RPO（数据恢复点目标） | RTO（恢复时间目标） | 成本 | 复杂度 |
|---------|-------------------|-----------------|------|-------|
| 单集群，单区域 | N/A（无容灾） | 1-6 小时 | 低 | 低 |
| 单集群，多可用区 | ~0（etcd 跨 AZ） | 分钟级 | 中 | 中 |
| 主备集群（同区域） | 1 分钟 | 5-15 分钟 | 高 | 高 |
| 主备集群（跨区域） | 5-15 分钟 | 15-30 分钟 | 很高 | 很高 |
| 双活/多活（多集群） | ~0 | ~0 | 极高 | 极高 |

**推荐策略**：

```
中等规模生产（RPO < 5 分钟，RTO < 30 分钟）：
  1. 集群内：3 AZ 部署控制平面（etcd 跨 AZ）
  2. etcd 每 30 分钟快照到 S3（跨区域复制）
  3. 应用层：Deployment 多副本分布在不同 AZ

关键业务（RPO ≈ 0，RTO < 5 分钟）：
  1. 主集群（Primary）：3 AZ 部署
  2. 备集群（Standby）：同区域不同 VPC，实时同步应用状态
  3. 数据库：跨区域同步（如 Aurora Global Database / Spanner）
  4. DNS 切换：运行健康检查，自动切流到备集群
```

## K8s 集群成本优化策略

K8s 集群成本通常占云费用的 40-60%，是最值得投入精力优化的领域。

### 成本优化策略全景

```
K8s 集群成本优化金字塔：

               ┌──────────┐
               │ 需求侧   │  ← 最有价值
               │ 优化      │
               ├──────────┤
               │ 调度侧   │
               │ 优化      │
               ├──────────┤
               │ 节点侧   │  ← 最常操作
               │ 优化      │
               ├──────────┤
               │ 采购侧   │  ← 最直接省钱
               │ 优化      │
               └──────────┘
```

| 优化层次 | 策略 | 典型节省 | 实施难度 |
|---------|------|---------|---------|
| **采购侧**（最直接） | 预留实例 / Savings Plan<br>Spot 实例 | 30-60%<br>60-90% | 低 |
| **节点侧** | 节点池拆分（按需 + Spot）<br>节点类型右移（更大的实例效率更高）<br>Cluster Autoscaler | 15-40% | 中 |
| **调度侧** | Pod 合理 Requests/Limits<br>Binpack 调度策略<br>节点亲和性 + 反亲和性 | 10-30% | 中-高 |
| **需求侧**（最有价值） | HPA / VPA 动态伸缩<br>闲置资源回收<br>无用的 Namespace/资源清理 | 20-50% | 高 |

### 节点池设计

#### Spot 实例 + 按需实例混合策略

```
混合节点池架构：

┌──────────────────────────────────────────┐
│ 节点池 A：按需实例（基准容量，3 节点）      │
│  ├─ 关键服务（API、DB、Auth）              │
│  └─ 不可被中断的 Prometheus / 监控栈       │
│                                            │
│ 节点池 B：Spot 实例（弹性容量，5-15 节点）   │
│  ├─ 非关键服务（后台 Worker、Batch Job）    │
│  ├─ 可容错服务（短时间内重调度可接受）       │
│  └─ 有 PodDisruptionBudget 保护            │
│                                            │
│ 节点池 C：GPU Spot 实例（训练集群）         │
│  ├─ 非实时训练任务                          │
│  ├─ 支持 checkpoint 恢复                   │
│  └─ 中断时自动迁移到按需节点池               │
└──────────────────────────────────────────┘
```

**实现示例**：

```bash
# AWS EKS 节点组配置（eksctl）

# 按需节点池（关键服务）
cat > nodegroup-on-demand.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production
  region: ap-northeast-1
managedNodeGroups:
  - name: on-demand-critical
    instanceType: m6i.xlarge
    minSize: 3
    maxSize: 10
    desiredCapacity: 3
    spot: false
    labels:
      node-type: on-demand
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"

  - name: spot-general
    instanceType: m6i.xlarge
    minSize: 5
    maxSize: 30
    desiredCapacity: 5
    spot: true
    spotAllocationStrategy: "capacity-optimized"
    labels:
      node-type: spot
    taints:
      - key: "spot"
        value: "true"
        effect: "PreferNoSchedule"
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
EOF

eksctl create nodegroup -f nodegroup-on-demand.yaml
```

**Pod 调度配置**：

```yaml
# 关键服务 → 绑定按需节点
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  template:
    spec:
      nodeSelector:
        node-type: on-demand
      tolerations: []

# 非关键服务 → 优先 Spot，容忍按需（fallback）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: background-worker
spec:
  template:
    spec:
      nodeSelector:
        node-type: spot
      tolerations:
        - key: "spot"
          value: "true"
          effect: "PreferNoSchedule"
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: "topology.kubernetes.io/zone"
          whenUnsatisfiable: ScheduleAnyway
```

### HPA / VPA 配置

#### Horizontal Pod Autoscaler（水平扩缩容）

```yaml
# HPA 示例：基于 CPU + 内存 + 自定义指标
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65    # CPU 目标利用率 65%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75    # 内存目标利用率 75%
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"      # 每秒请求数目标
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 缩容稳定窗口：5 分钟
      policies:
        - type: Percent
          value: 10                    # 每次最多缩 10%
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0    # 扩容立即执行
      policies:
        - type: Percent
          value: 100                   # 每次最多翻倍
          periodSeconds: 15
```

#### Vertical Pod Autoscaler（垂直扩缩容）

VPA 自动调整 Pod 的 CPU/内存 Requests，消除人工估算误差。

```yaml
# VPA 推荐模式（推荐生产使用模式）
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  updatePolicy:
    updateMode: "Off"           # 仅推荐，不自动更新
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: "50m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4"
          memory: "8Gi"
        controlledResources: ["cpu", "memory"]
```

**VPA 三种模式**：

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `Off` | 仅生成推荐值（不自动修改） | 生产首选：先观察推荐的准确性 |
| `Auto` | 自动更新 Pods（需 Pod 重建） | 可重启的服务：后台 Worker |
| `Initial` | 仅在创建时设置（之后不变） | Jamstack 或不可重启的服务 |

### 实际优化效果

#### 资源超卖配置

```yaml
# 节点池开启资源超卖（使用 Descheduler + Node Resource Manager）
# 原理：实际 Pod 使用量远小于 Requests，超卖可以提高资源利用率

# 示例：节点 8 vCPU，32GB RAM
# 正常配置：Requests 总和不可超过节点容量
# 超卖配置：允许 Requests 总和 = 节点容量的 150-200%
# 风险控制：通过监控确保实际使用不超过 80% 节点容量，超过则触发 Cluster Autoscaler 扩容

# 使用 Descheduler 定期平衡
apiVersion: v1
data:
  policy.yaml: |
    apiVersion: "descheduler/v1alpha2"
    kind: "DeschedulerPolicy"
    strategies:
      "LowNodeUtilization":
        enabled: true
        params:
          nodeResourceUtilizationThresholds:
            thresholds:
              cpu: 30
              memory: 30
            targetThresholds:
              cpu: 60
              memory: 60
```

### 成本优化实践检查清单

| 检查项 | 操作 | 预估节省 |
|-------|------|---------|
| ☐ 启用 Cluster Autoscaler | 确保节点自动扩缩 | 15-25% |
| ☐ 评估 Spot 实例比例 | 将 40-70% 的非关键负载迁移到 Spot | 30-50% |
| ☐ 检查 Pod Requests 合理性 | 减少 50% 以上（实际使用远小于 Request）的 Pod | 10-20% |
| ☐ 启用 VPA（推荐模式） | 收集 7 天数据后分析推荐值 | 5-15% |
| ☐ 配置 HPA | 确保 3 个工作日的指标历史 | 10-20% |
| ☐ 清理未使用的资源 | 检查 24h 空闲的 Pod/Service/PVC | 2-5% |
| ☐ 使用预留实例 / Savings Plan | 覆盖 40-70% 的按需实例 | 30-50% |
| ☐ 考虑 ARM 实例（Graviton） | 如果应用兼容 | 20-40% |
| ☐ 检查存储成本 | 让 PVC 使用正确 StorageClass | 5-15% |
| ☐ 检查网络出站成本 | 减少跨区域/跨可用区的数据传输 | 5-20% |

## K8s 核心操作速查

```bash
kubectl get pods                    # 查看 Pod
kubectl get pods -o wide            # 查看 Pod + IP + 节点
kubectl get all                     # 查看所有资源
kubectl describe pod <name>         # 详细状态
kubectl logs -f pod/<name>          # 查看日志
kubectl exec -it <pod> -- /bin/sh   # 进入容器
kubectl port-forward pod/<name> 8080:80  # 端口转发
kubectl apply -f deployment.yaml    # 声明式创建/更新
kubectl delete -f deployment.yaml   # 删除
kubectl rollout status deploy/web   # 查看滚动更新状态
kubectl rollout undo deploy/web     # 回滚到上个版本
kubectl scale deploy/web --replicas=5  # 手动扩缩容
```

## K8s 的运维心智模型

```
┌─────────────────────────────────────┐
│         声明式                      │
│   "我想要的最终状态"                │
│   replicas: 5                      │
│   image: nginx:1.25                │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│         控制循环                    │
│   Controller Manager + Scheduler   │
│   不断将"当前状态" 收敛为"期望状态"   │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│         自愈                        │
│   Pod 挂了 → 自动重新创建           │
│   节点宕了 → 迁移 Pod              │
│   镜像更新了 → 滚动替换             │
└─────────────────────────────────────┘
```

## 生产案例：某在线教育平台的K8s成本优化案例

### 业务背景

**公司概况**：
- 某头部在线教育平台（日活 300 万+，技术团队 120 人）
- 核心业务：在线直播授课、作业批改、题库系统、AI 学习推荐
- 基础设施规模：120 个微服务，部署在 3 个 K8s 集群（直播、业务、AI）

**迁移前成本状况**：

| 维度 | 说明 |
|------|------|
| 月度云成本总额 | $45,000 |
| 其中 K8s 相关成本 | $38,000（占 84%） |
| Pod 总量 | 约 1,800 个（3 个集群总计） |
| Worker 节点数 | 约 60 个（按需实例，m5.xlarge / m5.2xlarge） |
| 月均节点资源利用率 | CPU: 15% / 内存: 28% |

### 核心痛点

| 痛点 | 严重程度 | 数据支撑 |
|------|---------|---------|
| 资源利用率极低 | 🔴 严重 | 平均 CPU 15%，大量节点 CPU 低于 10% |
| Pod Requests 过大 | 🔴 严重 | 约 70% 的 Pod 的 Requests 是实际使用的 3-5 倍 |
| 无自动伸缩 | 🟠 中 | 只有 10% 的 Deployment 配置了 HPA |
| 节点配置一刀切 | 🟠 中 | 所有节点池用同类型实例，没有 Spot 实例 |
| 无成本归属 | 🟢 低 | 不知道哪个团队"花钱最多" |
| 无资源清理机制 | 🟠 中 | 存在 30+ 个已废弃服务的 Deployment |

### 具体分析

```yaml
典型 Pod 的资源配置分析（来自实际数据）：

服务: user-service (用户服务)
  Requests:
    cpu: 500m (实际 P95: 45m，超配 11 倍)
    memory: 1Gi (实际 P95: 128Mi，超配 8 倍)
  Limits: 未设置

服务: live-stream-gateway (直播网关)
  Requests:
    cpu: 1 (实际 P95: 320m，超配 3 倍)
    memory: 2Gi (实际 P95: 512Mi，超配 4 倍)
  Limits: 未设置

问题总结：
  - 大部分服务 Request 设置凭"经验估值"
  - 没有资源监控配套
  - 开发为了"稳定"故意多申请资源
  - 没有成本意识（"反正公司出钱"）
```

### 优化方案

#### 四步优化策略

```
Phase 1 — 资源分析洞察（第 1-2 周）
  ├─ 部署 Prometheus + Grafana + Kubecost
  ├─ 分析所有 Pod 的 30 天实际资源使用（P50/P95/P99）
  ├─ 列出 Top 20 "浪费"的服务（超配最严重的）
  └─ 输出：每个服务的推荐 Requests 值

Phase 2 — VPA 推荐 + 手动调整（第 3-4 周）
  ├─ 对所有无状态 Deployment 安装 VPA（推荐模式）
  ├─ 每 24 小时生成推荐值
  ├─ 开发团队审核推荐值是否合理
  ├─ 分批调整 Requests（每天 10-20 个服务）
  └─ 目标：将超配倍数从平均 5x 降到 1.5x

Phase 3 — 节点池改造（第 5-6 周）
  ├─ 创建 3 个节点池：
  │   ├─ 按需池（关键服务，3 节点）
  │   ├─ Spot 池（非关键服务，10-20 节点）
  │   └─ GPU 池（AI 模型推理，2-4 节点）
  ├─ 配置 Cluster Autoscaler（5-30 节点范围）
  ├─ 对非关键服务添加 PodDisruptionBudget
  └─ 目标：Spot 实例占 60% 工作负载

Phase 4 — HPA + 持续优化（第 7-8 周）
  ├─ 对 50+ 个服务配置 HPA（基于 CPU + 内存）
  ├─ 关键服务配置 Prometheus 自定义指标
  ├─ 搭建成本 Dashboard（按 Namespace / 团队 / 服务）
  ├─ 建立月度成本 Review 制度
  └─ 目标：CPU 利用率 > 40%，系统稳定
```

#### VPA 推荐实施

```bash
# 安装 VPA（适用人群：Admission Controller 已启用）
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vpa-1.0.0/vpa.yaml

# 为所有 Deployment 生成 VPA（推荐模式）
for deploy in $(kubectl get deploy -n default -o name | cut -d/ -f2); do
  cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: ${deploy}-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${deploy}
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
      maxAllowed:
        cpu: "4"
        memory: "8Gi"
EOF
done

# 查看 VPA 推荐值
kubectl get vpa user-service-vpa -n default -o yaml | grep -A 10 recommendation
# recommendation:
#   containerRecommendations:
#   - containerName: user-service
#     target:
#       cpu: 85m       # 原来 500m → 推荐 85m
#       memory: 262144k # 原来 1Gi → 推荐 256Mi
#     lowerBound:
#       cpu: 40m
#       memory: 128Mi
#     upperBound:
#       cpu: 350m
#       memory: 512Mi

# 根据 VPA 推荐手动调整 Deployment（分批进行）
```

#### 节点池配置

```hcl
# Terraform 式的节点池配置（示例用 eksctl 语法）

# 按需节点池（关键服务基准）
nodegroup "on-demand-critical" {
  instance_types = ["m6i.xlarge", "m6a.xlarge"]
  min_size       = 3
  max_size       = 10

  labels = {
    "node.kubernetes.io/lifecycle" = "on-demand"
  }
}

# Spot 节点池（弹性扩展）
nodegroup "spot-general" {
  instance_types      = ["m6i.xlarge", "m6a.xlarge", "c6i.xlarge"]
  spot_allocation_strategy = "capacity-optimized"
  min_size            = 5
  max_size            = 30

  labels = {
    "node.kubernetes.io/lifecycle" = "spot"
    "k8s.spot.eligibility"        = "true"
  }

  taints = [
    {
      key    = "spot"
      value  = "true"
      effect = "PreferNoSchedule"
    }
  ]
}
```

#### HPA 配置示例

```yaml
# 核心服务：多层次 HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: live-stream-gateway-hpa
  namespace: live
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: live-stream-gateway
  minReplicas: 5
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Pods
      pods:
        metric:
          name: live_connections
        target:
          type: AverageValue
          averageValue: "500"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 200       # 高峰期快速扩容
          periodSeconds: 30
```

### 关键指标前后对比

| 指标 | 优化前 | 优化后 | 改善幅度 |
|------|--------|--------|---------|
| 月度 K8s 成本 | $38,000 | $22,000 | 42% ↓ |
| 总云成本 | $45,000 | $26,000 | 42% ↓ |
| CPU 平均利用率 | 15% | 52% | 3.5 倍 ↑ |
| 内存平均利用率 | 28% | 58% | 2.1 倍 ↑ |
| Spot 实例占比 | 0% | 55% 工作负载 | 新增 |
| Cluster Autoscaler 覆盖 | 无 | 全集群 | — |
| 节点数量 | ~60（固定） | 15-40（动态） | 33-75% ↓ |
| HPA 覆盖率 | 10% | 70% | 7 倍 |
| VPA 覆盖率 | 0% | 100%（推荐模式） | 新增 |
| 废弃服务资源 | 30+ 个 Deployment | 全部清理 | — |
| 月度资源浪费（超配） | ~$8,000 | ~$1,500 | 81% ↓ |
| SLA 违约 | 0% | 0%（无降级） | — |
| P99 响应延迟 | 120ms | 95ms | 21% ↓（受益于合理资源配置） |

### 经验教训

1. **过度配置比配置不足更可怕**
   - 开发团队为"稳定性"设置过大的 Requests，导致大量资源闲置
   - 教训：资源设置应有数据支撑（P95/P99 实际用量），而非凭感觉

2. **VPA 推荐依赖准确的历史数据**
   - 初期 VPA 推荐值波动很大，因为只收集了 3 天的数据
   - 教训：至少收集 7-14 天（涵盖周末与工作日峰值）的数据后再采纳推荐

3. **Spot 实例需要 PodDisruptionBudget（PDB）**
   - 一次 AWS EC2 大规模中断影响 Spot 实例池，30% 的非关键 Pod 被回收
   - 由于没有 PDB，回收时 Pod 几乎同时被拉走，导致短暂不可用
   - 教训：Spot 实例必须配置 PDB，确保最多 30% 的非关键 Pod 同时不可用

4. **HPA 太快缩容导致"踩踏效应"**
   - 流量低谷期 HPA 快速缩容，结果流量稍有回升就立刻扩容
   - 反复扩缩不仅影响体验，还增加了节点 Auto Scaling 的调用频率
   - 教训：设置 `stabilizationWindowSeconds` 为 5-10 分钟，缩容更保守

5. **成本归属必须明确**
   - 使用 Kubecost 或类似工具按 Namespace / Label 拆分成本
   - 每个团队看到自己的成本数据，自然就有优化动力
   - 教训：没有成本归属的优化是不可持续的，要建立"谁用谁付"的机制

6. **过度优化也有风险**
   - 某服务 Pod Requests 从 1CPU/2Gi 降到 100m/256Mi 后，偶发流量高峰频繁触发 OOM
   - 教训：优化不是越极端越好，设置的 Requests 至少应该覆盖 P95 使用量，留有一定的余量

## 本章小结

| 要点 | 说明 |
|------|------|
| K8s 本质 | 容器编排平台，解决大规模容器管理的集群问题 |
| 核心架构 | Control Plane（管控）+ Worker Node（计算） |
| 最小单元 | Pod（一个或多个容器的组合）|
| 发行版选型 | K3s 适合边缘，EKS/GKE/AKS 适合云原生，OpenShift 适合合规 |
| Pod 资源管理 | CPU 建议延迟敏感型 1:1，内存始终 1:1，基于历史数据设置 |
| 网络方案 | Calico BGP 适合大规模，Cilium eBPF 性能最强，Flannel 最简单 |
| 高可用设计 | 3 AZ 分布控制平面，etcd 30 分钟定期备份，跨区域 DR |
| 成本优化 | VPA + HPA + Spot 实例 + 节点池拆分 = 42% 成本降低 |
| Deployment | 无状态应用的声明式管理 |
| Service | 稳定的网络入口 |
| Ingress | HTTP 路由 |
| 配置管理 | ConfigMap（非敏感）、Secret（敏感）|
| 声明式心智 | 描述期望状态，K8s 控制循环自动收敛 |

