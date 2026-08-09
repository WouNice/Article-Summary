# 第11章：GitOps：以 Git 为唯一真相源

## 本章学习目标

- 理解 GitOps 的核心原则与价值主张
- 掌握 GitOps 的工作模式——Pull vs Push
- 了解 ArgoCD 和 Flux 的核心机制
- 能够设计基于 GitOps 的多环境管理方案

## GitOps 是什么

### 定义

> **GitOps** 是一种以 Git 仓库作为**唯一真相源**的运维模型。整个系统的期望状态声明在 Git 中，并通过自动化工具持续将实际状态收敛为期望状态。

### 核心原则

1. **声明式**：整个系统的期望状态通过声明式配置描述
2. **版本化且不可变**：所有状态存储在 Git 中，每次变更都有完整审计历史
3. **自动同步**：变更被合并到 Git 后，系统自动收敛到新状态
4. **软件代理**：集群中运行一个 Agent（Operator），负责确保集群状态与 Git 一致

```
                    Git 仓库
                   (唯一真相源)
                      │
                      │ commit / PR
                      ▼
               ┌──────────────┐
               │  GitOps      │  ← 检测到 Git 变更
               │  Operator    │    对比当前状态与期望状态
               │  (ArgoCD)    │    执行同步
               └──────┬───────┘
                      │
                      ▼
               ┌──────────────┐
               │ K8s Cluster  │  ← 始终与 Git 保持同步
               └──────────────┘
```

## GitOps vs 传统 CI/CD

### 传统部署模式（Push）

```
CI/CD 工具              K8s Cluster
┌─────────┐              ┌─────────┐
│ Jenkins │──kubectl──→│  Pod A  │
│ Apply   │────apply───→│  Pod B  │
└─────────┘              └─────────┘
```

**问题**：
- CI/CD 工具需要访问集群的 kubeconfig —— 安全风险
- 谁执行了哪个 `kubectl`？—— 审计困难
- 环境漂移：有人手动 kubectl edit 了，但没人知道

### GitOps 模式（Pull）

```
Git 仓库                 K8s Cluster
┌─────────┐              ┌────────────┐
│ manifests│  ←Pull─────│ ArgoCD     │
│ + config│  ←对比状态  │ Operator   │
└─────────┘              │  ├── Apply │
                         │  └── Sync  │
                         └────────────┘
```

**优势**：
| 维度 | Push 模式 | GitOps (Pull) 模式 |
|------|----------|-------------------|
| 安全 | CI 工具需要集群凭证 | 集群 Agent 拉取 Git，无需外放凭证 |
| 审计 | CI 工具日志可能不完整 | 每个变更 = 一个 Git Commit |
| 漂移 | 容易被手动操作覆盖 | Agent 持续检测并修正漂移 |
| 回滚 | `kubectl rollout undo` | `git revert` 即可 |
| PR 流程 | 无法 CI/CD 生效前评审 | 先评审 manifests 再合并 |

## 核心工具：ArgoCD

### ArgoCD 架构

```
┌─────────────────────────────────────┐
│          ArgoCD Control Plane        │
│  ┌──────────┐ ┌────────┐ ┌───────┐ │
│  │ API      │ │ Repo   │ │  Application│
│  │ Server   │ │ Server │ │ Controller │
│  └──────────┘ └────────┘ └───────┘ │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│       Target K8s Cluster(s)         │
│   (auto-sync + drift detection)     │
└─────────────────────────────────────┘
```

### ArgoCD Application 示例

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service
  namespace: argocd
spec:
  project: default

  # 源：Git 仓库
  source:
    repoURL: https://github.com/team/my-service.git
    targetRevision: main    # 分支 / Tag / Commit SHA
    path: k8s/overlays/prod # 清单文件路径

  # 目标：部署到哪个集群
  destination:
    server: https://kubernetes.default.svc  # 当前集群
    namespace: prod

  # 同步策略
  syncPolicy:
    automated:
      prune: true       # 删除 Git 中不存在的资源
      selfHeal: true    # 检测并修复漂移
    syncOptions:
    - CreateNamespace=true
```

### ArgoCD 同步策略

| 策略 | 说明 | 推荐 |
|------|------|------|
| **Manual** | 人工点击 Sync | 生产环境 |
| **Auto Sync** | 检测到 Git 变更自动同步 | 开发/测试环境 |
| **Auto + SelfHeal** | 自动同步 + 修复手动修改 | 严格 GitOps |
| **Auto + Prune** | 自动同步 + 删除 Git 中移除的资源 | 谨慎使用（可能误删） |

### ArgoCD 的 UI 心智模型

```
Application
├── LIVE STATE（集群当前状态）
│   └── Deployment / Service / ConfigMap ...（与实际一致）
├── DESIRED STATE（Git 中的配置）
│   └── Deployment / Service / ConfigMap ...（声明式）
├── DIFF（差异对比）
│   ├── OutOfSync（不一致）
│   └── Synced（一致）
└── Sync Status
    ├── Healthy（全部正常）
    ├── Degraded（部分异常）
    └── Progressing（正在同步）
```

### ArgoCD vs Flux 深度对比

ArgoCD 和 Flux 是 GitOps 生态中最主流的两个工具，但设计理念和侧重点不同。以下是全面的多维对比：

| 对比维度 | ArgoCD | Flux v2 |
|---------|--------|---------|
| **项目归属** | CNCF 毕业项目（捐赠自 Intuit） | CNCF 毕业项目（原 Weaveworks） |
| **首次发布** | 2018 | 2021（Flux v2，从 v1 重写） |
| **架构设计** | 中心化控制平面 + API Server | 无状态 CRD Operator（无 API Server） |
| **安装复杂度** | 较复杂（CRDs + API Server + Dex + Config） | 简单（单 Operator 部署） |
| **Git 源支持** | Git / Helm / OCI | Git / Helm / OCI / Bucket / S3 |
| **配置管理** | Kustomize 原生 / Helm / Jsonnet | Kustomize 原生 / Helm |
| **多集群支持** | ✅ 原生支持（一个控制平面管理多集群） | ✅ 支持（Kustomization 跨集群） |
| **RBAC/SSO** | 强（内置 Dex + RBAC Project） | 弱（依靠 K8s RBAC，需额外组件） |
| **UI/UX** | ✅ 成熟 Web UI + CLI（argocd） | ⚠️ CLI（flux）+ 可选 Web UI（Weave GitOps） |
| **漂移检测** | ✅ 默认 3 分钟轮询 | ✅ 可配置间隔（默认轮询） |
| **同步策略** | Manual / Auto / Sync Windows / Sync Waves | Auto / Manual（依赖 Kustomize/Helm） |
| **Secret 管理** | 集成 SealedSecrets / Vault（通过 Manifests） | 集成 SOPS（内置支持） |
| **Image Updater** | ✅ ArgoCD Image Updater | ✅ Image Automation Controller（内置） |
| **Webhook 性能** | 支持（ArgoCD Webhook Handler） | 支持（Notification Controller） |
| **社区活跃度** | ⭐⭐⭐⭐⭐（CNCF 最受欢迎 GitOps 工具） | ⭐⭐⭐⭐（稳步增长） |
| **GitHub Stars** | 17K+ | 8K+ |
| **学习曲线** | 中等（概念多但文档完善） | 低（简洁设计） |

#### ArgoCD vs Flux：决策建议

```
选择 ArgoCD 的场景：
├─ 需要成熟的多集群管理（200+ 集群全在一个控制平面）
├─ 需要企业级 RBAC/SSO（LDAP/OIDC/Dex 集成）
├─ 需要丰富的 UI 和 Operation 团队使用
├─ 需要 Sync Waves 控制资源创建顺序
├─ 团队已经使用 Kustomize 且有复杂 overlay 结构
└─ 典型代表：大型企业、金融、多业务线

选择 Flux 的场景：
├─ 追求简洁架构（无 API Server，纯 CRD）
├─ 只需要单一集群或少数集群
├─ 团队 DevOps 能力强，偏好 CLI 而非 UI
├─ 需要原生 SOPS 集成（ArgoCD 需要额外配置）
├─ 需要直接使用 S3/Bucket 作为源（非 Git 场景）
├─ 需要自动镜像更新策略（内置 Image Automation）
└─ 典型代表：中小团队、云原生初创公司、平台工程

组合使用：
├─ ArgoCD 做集群部署 + Flux 做应用级自动镜像更新
└─ 不推荐：两个工具做同一件事，增加运维复杂度
```

**单集群 vs 多集群成本对比**：

| 场景 | ArgoCD 部署成本 | Flux 部署成本 | 建议 |
|------|----------------|--------------|------|
| 1-5 个集群 | 中等（控制平面开销） | 低 | Flux（减少运维） |
| 5-20 个集群 | 中等（控制平面共享） | 中等 | 取决于团队技能 |
| 20-200+ 集群 | 低（控制平面复用率高） | 高（需额外编排） | ArgoCD + ApplicationSet |

## 多环境管理策略

### 方案一：同一仓库不同路径

```
my-service/
├── k8s/
│   ├── base/              # 公共配置（Kustomize base）
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── overlays/
│   │   ├── dev/           # dev 环境覆写
│   │   ├── staging/       # staging 环境覆写
│   │   └── prod/          # prod 环境覆写
```

**Pros**：单仓库便于管理，公共配置复用
**Cons**：Prod 的访问权限可能不能隔离

### 方案二：不同分支

```
main 分支 ──→ 开发环境
staging 分支 ──→ 预发布环境
release/* 分支 ──→ 生产环境
```

**Pros**：通过分支权限控制环境
**Cons**：Prod 配置变更了但没合并回 main → 分叉

### 方案三：不同仓库（推荐用于大型项目）

```
gitops-infra/
├── envs/
│   ├── dev/        → 对应 dev 集群，main 分支
│   ├── staging/    → 对应 staging 集群，main 分支
│   └── prod/       → 对应 prod 集群，main 分支
                       但通过 CODEOWNERS 限制谁可以合入 prod/
```

## 端到端 CI + GitOps 工作流设计

### GitOps 对 CI 流程的影响

传统 CI/CD 中，CI 和 CD 是一体的——CI 构建完毕后直接 `kubectl apply` 推送到集群。GitOps 将这两者分离：**CI 负责"Build"，GitOps 负责"Sync"**。

```
传统模式（Push）：
代码提交 → CI 构建镜像 → CI 直接 kubectl apply 到 K8s
                                        ↑ CI 需要集群凭证（安全风险）

GitOps 模式（Pull）：
代码提交 → CI 构建镜像 → CI 更新 Git 仓库中的 manifests → ArgoCD 感知变更 → 拉取并同步到 K8s
                          ↑                                 ↑
                    CI 只需要 Git 写权限             集群 Agent 从 Git 拉取（安全）
```

### 完整端到端工作流

```
步骤                                   谁负责            产物/输出
────────────────────────────────────────────────────────────────────
1. 开发 Push 代码到 PR               开发者            Commit SHA: abc123
2. CI 运行测试 + SAST + SCA          CI Pipeline      ✅ 测试通过
3. CI 构建镜像并 Push 到 Registry     CI Pipeline      registry/my-app:abc123
4. CI 自动更新 Git 仓库的 manifests    CI Pipeline      Git Commit: "更新镜像 Tag 为 abc123"
   （通过 Image Updater / 或 CI 直接 commit）
5. ArgoCD/Flux 检测到 manifests 变更  GitOps Operator  Git 差异对比
6. 同步到 K8s 集群                    GitOps Operator  K8s 资源更新
```

### 完整流水线 YAML 示例

```yaml
# .github/workflows/ci-cd-gitops.yaml
name: CI + GitOps Build & Deploy

on:
  push:
    branches: [main]
    paths-ignore: ['k8s/**']  # 避免 gitops 循环触发

env:
  REGISTRY: registry.example.com
  IMAGE_NAME: my-app
  GITOPS_REPO: team/gitops-infra
  GITOPS_BRANCH: main

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
    outputs:
      image_tag: ${{ steps.vars.outputs.image_tag }}

    steps:
    - name: Checkout Source
      uses: actions/checkout@v4

    - name: Set Image Tag
      id: vars
      run: echo "image_tag=${GITHUB_SHA::8}-$(date +%Y%m%d%H%M%S)" >> $GITHUB_OUTPUT

    - name: Build & Push Docker Image
      run: |
        docker build -t $REGISTRY/$IMAGE_NAME:${{ steps.vars.outputs.image_tag }} .
        docker push $REGISTRY/$IMAGE_NAME:${{ steps.vars.outputs.image_tag }}

    - name: Scan Image for Vulnerabilities
      run: |
        trivy image --severity CRITICAL --exit-code 1 $REGISTRY/$IMAGE_NAME:${{ steps.vars.outputs.image_tag }}

  update-manifests:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Checkout GitOps Repo
      uses: actions/checkout@v4
      with:
        repository: ${{ env.GITOPS_REPO }}
        token: ${{ secrets.GITOPS_PAT }}
        path: gitops

    - name: Update Image Tag in Manifests
      run: |
        cd gitops
        # 使用 Kustomize 的 edit 命令更新镜像 Tag
        kustomize edit set image $REGISTRY/$IMAGE_NAME:.*=$REGISTRY/$IMAGE_NAME:${{ needs.build.outputs.image_tag }}

        # 如果使用 Helm，则更新 values 文件
        # sed -i "s|tag:.*|tag: ${{ needs.build.outputs.image_tag }}|g" values-prod.yaml

    - name: Commit and Push
      run: |
        cd gitops
        git config user.name "CI Bot"
        git config user.email "ci-bot@example.com"
        git add .
        git commit -m "chore: update $IMAGE_NAME image tag to ${{ needs.build.outputs.image_tag }}"
        git push
      env:
        GIT_AUTHOR_NAME: "CI Bot"
        GIT_AUTHOR_EMAIL: "ci-bot@example.com"

  # ========== GitOps 阶段：Git Push 后自动触发 GitOps Operator 同步 ==========
  # 以下由 ArgoCD/Flux 在集群内自动完成（不需要 CI 步骤）
  #
# ArgoCD/Flux 检测到 gitops-infra 仓库的变更
# 对比当前 K8s 状态与 Git 中声明的状态
# 如果发现差异（新的镜像 Tag），执行同步
# 更新 Deployment 的镜像版本，触发 Rolling Update
# 持续监控直到新 Pod 达到 Ready 状态
```

### Tag 策略决策

在 GitOps 中，镜像 Tag 策略直接影响部署的可追溯性和自动化升级的效率。

| Tag 策略 | 格式示例 | 可读性 | 可追溯性 | 自动化升级 | 调试难度 |
|---------|---------|--------|---------|-----------|---------|
| **SemVer** | `v1.2.3` | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | 低 |
| **Commit SHA** | `abc1234` | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 低（能精准映射代码版本） |
| **时间戳** | `20230915-1430` | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | 中（仅知道什么时候构建） |
| **SemVer+SHA** | `v1.2.3-abc1234` | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 低（推荐策略） |
| **Git Branch+SHA** | `main-abc1234` | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 中 |
| **UUID** | `a1b2c3d4-e5f6` | ⭐ | ⭐⭐⭐ | ⭐⭐ | 高（不可人类阅读） |

**推荐策略**：
```yaml
# 策略一：Commit SHA（简洁，适合自动化）
image_tag: ${GITHUB_SHA::8}  # fa3b2c1e

# 策略二：SemVer + Commit SHA（可读+可追溯，推荐生产环境）
image_tag: v1.2.3-fa3b2c1e

# 策略三：语义化时间戳（适合高频发布）
image_tag: 20240315-1430-${GITHUB_RUN_NUMBER}
```

> **关键原则**：Tag 必须**不可变**（Immutable）。绝不复用同一个 Tag 推送不同内容——这是镜像安全的基础。使用 Commit SHA 作为 Tag 天然保证了不可变性。

## Kustomize 与 Helm

### Kustomize（内建 ArgoCD）

Kustomize 是**模板免安装**的配置管理工具，通过 overlay 实现环境差异化。

```yaml
# kustomization.yaml (base)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml

# kustomization.yaml (prod overlay)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../base
patches:
  - path: increase-replicas.yaml  # 修改副本数为 10
```

### Helm（K8s 包管理器）

Helm 通过模板引擎实现参数化部署。

```yaml
# values-prod.yaml
replicaCount: 10
resources:
  limits:
    memory: "2Gi"
ingress:
  enabled: true
  hosts:
    - app.example.com
```

**ArgoCD 支持 Helm**：
```yaml
source:
  path: charts/my-app
  repoURL: ...
  helm:
    valueFiles:
      - values-prod.yaml
```

| 工具 | 适用场景 | 学习曲线 |
|------|---------|---------|
| **Kustomize** | K8s 原生，无模板语法 | ⭐ |
| **Helm** | 应用打包、参数化部署 | ⭐⭐⭐ |

## GitOps 在非 K8s 环境中的应用

虽然 GitOps 最初和 K8s 深度绑定，但其"声明式配置 + Git 版本控制 + 自动收敛"的理念也适用于基础设施管理。

### Terraform Cloud/Enterprise + GitOps

Terraform + GitOps 是 IaC 领域的经典组合：

```yaml
# Terraform Cloud VCS 驱动的 GitOps 工作流

# 仓库结构
iac/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       └── terraform.tfvars
├── modules/
│   ├── vpc/
│   ├── ecs/
│   └── rds/
└── .github/
    └── workflows/
        └── terraform-plan-apply.yaml
```

**Terraform Cloud GitOps 工作流**：

```
PR 创建 → Terraform Plan 自动运行 → 人工评审 ✅ → 合并到 main
  │                                       │                │
  ▼                                       ▼                ▼
Terraform Cloud                          Terraform        Terraform
Run Plan                                 Plan（更新）      Apply（执行）
```

```yaml
# Terraform Cloud Run Trigger（Git Push 自动触发 Apply）
resource "tfe_workspace" "prod" {
  name         = "infra-prod"
  organization = "myorg"
  # 当 main 分支变更时自动触发 Run
  trigger_patterns = ["environments/prod/*"]

  # 自动 Apply（需更严格的 Code Review 机制）
  auto_apply = false  # 生产环境建议人工确认
}
```

### Crossplane：面向云资源的 GitOps

Crossplane 是 CNCF 项目，将 GitOps 模式扩展到云资源的声明式管理：

```yaml
# Crossplane 声明式创建 RDS 实例
apiVersion: database.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: orders-db
  annotations:
    # GitOps 拥抱：资源由 Git 管理
    crossplane.io/external-name: orders-db-prod
spec:
  forProvider:
    region: us-east-1
    dbInstanceClass: db.t3.medium
    engine: postgres
    engineVersion: "15"
    allocatedStorage: 100
    masterUsername: admin
    # 从 Vault 动态获取密码
    masterPasswordSecretRef:
      name: db-password
      namespace: crossplane-system
      key: password
  writeConnectionSecretToRef:
    name: orders-db-conn
    namespace: prod
```

**Terraform vs Crossplane 对比**：

| 对比维度 | Terraform + GitOps | Crossplane |
|---------|-------------------|-----------|
| **声明式** | HCL 语言 | K8s CRD Yaml |
| **状态管理** | Terraform State（有状态文件） | K8s etcd + Status 字段 |
| **Drift 检测** | 每次 `terraform plan` 时 | K8s 控制器持续调和 |
| **多环境** | Workspace/Directory 隔离 | Namespace + ProviderConfig 隔离 |
| **K8s 集成** | 外部（通过 Terraform Operator） | 原生（K8s CRD 扩展） |
| **适用场景** | 传统 IaC + GitOps | K8s 原生团队的全栈 GitOps |

### Ansible + AWX/Ansible Tower

Ansible 通过 AWX/Ansible Tower 实现 GitOps 风格的自动化运维：

```yaml
# Ansible Playbook（配置管理与 GitOps）
---
- name: GitOps - Web Server Configuration
  hosts: webservers
  vars:
    app_version: "{{ git_branch | default('main') }}"
  tasks:
    - name: Pull latest application code
      git:
        repo: https://github.com/team/my-app.git
        dest: /opt/my-app
        version: "{{ app_version }}"

    - name: Restart application
      systemd:
        name: my-app
        state: restarted
```

**Terraform/Crossplane/Ansible 的 GitOps 适用范围**：

```
├─ 基础设施供应（VPC/子网/RDS/Redis）
│  └─ Terraform + GitOps / Crossplane
├─ 配置管理（OS 配置、软件包、agent 部署）
│  └─ Ansible + AWX / Ansible Automation Platform
├─ 应用部署（容器化应用、K8s）
│  └─ ArgoCD / Flux（K8s GitOps 原生）
└─ 统一管理
   └─ Terraform Cloud (orchestrate) + ArgoCD (K8s) + Ansible (VM)
```

## 密钥管理在 GitOps 中的最佳实践

GitOps 的核心是"Git 为唯一真相源"，但明文密钥**绝对不能提交到 Git**。以下是在 GitOps 中管理密钥的主流方案对比：

### 方案对比

| 方案 | 加密方式 | 易用性 | 自动化程度 | 审计能力 | Git 兼容 | 场景推荐 |
|------|---------|-------|-----------|---------|---------|---------|
| **SealedSecrets** | 非对称加密（Controller 持有私钥） | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ 可安全 commit | K8s 原生场景，最简单 |
| **External Secrets Operator** | 不加密，引用外部密钥（Vault/AWS/GCP） | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ YAML 中只存引用 | 已有密钥管理平台 |
| **SOPS + Mozilla** | Age/GPG/PGP 加密 YAML/JSON 文件 | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ 加密后可 commit | 非 K8s 场景，灵活 |
| **Vault Agent Sidecar** | Vault 动态凭据 + Sidecar 注入 | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ 无需密钥存 Git | 企业级动态凭据 |

### SealedSecrets 实践

```bash
# 安装 SealedSecrets Controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# 使用 kubeseal 加密 Secret
kubeseal --controller-namespace kube-system \
  --controller-name sealed-secrets-controller \
  --format yaml < secret.yaml > sealed-secret.yaml

# 加密后的 SealedSecret 可以安全提交到 Git
cat sealed-secret.yaml
```

```yaml
# 原始 Secret（⚠️ 不能提交到 Git！）
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
data:
  password: c3VwZXJzZWNyZXQ=   # "supersecret" 的 base64

# ✅ 加密后的 SealedSecret（可以安全提交到 Git）
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiYRYc2FgFUzLm7Q...  # 加密后的密文
```

### External Secrets Operator 实践

```yaml
# 定义 Secret Store（连接 Vault/AWS/GCP）
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: https://vault.example.com
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: external-secrets-role

# 定义 ExternalSecret（引用 Vault 中的密钥）
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h          # 每小时自动轮换
  secretStoreRef:
    name: vault-backend
  target:
    name: db-secret            # 输出的 K8s Secret 名称
  data:
  - secretKey: password        # K8s Secret 中的 key
    remoteRef:
      key: secret/data/production/db
      property: password
```

### SOPS + Age 实践

```bash
# 生成 Age 密钥
age-keygen -o age.key

# 创建 .sops.yaml 配置
cat > .sops.yaml << EOF
creation_rules:
  - path_regex: secrets/*.yaml
    age: age1abc123def...
EOF

# 加密文件（加密后的文件可安全提交到 Git）
sops --encrypt secrets/prod.yaml > secrets/prod.enc.yaml

# 解密查看
sops --decrypt secrets/prod.enc.yaml
```

```yaml
# 加密前的 secrets/prod.yaml
api_version: v1
kind: secret
data:
  db_password: supersecret123!

# 加密后的 secrets/prod.enc.yaml（可安全提交到 Git）
api_version: v1
kind: secret
data:
  db_password: ENC[AES256_GCM,data:3Q9X2P4R...]
sops:
  age:
    - recipient: age1abc123...
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSB...
        -----END AGE ENCRYPTED FILE-----
  lastmodified: "2024-03-15T10:30:00Z"
  mac: ENC[AES256_GCM,data:...]
```

### 方案综合对比

```
密钥管理选型决策：
├─ 场景：K8s 单集群，简单直接
│  └─ SealedSecrets（一键安装，无需额外基础设施）
├─ 场景：K8s 多集群，已有 Vault/AWS
│  └─ External Secrets Operator（统一密钥管理）
├─ 场景：非 K8s 环境（Terraform/Ansible/IaC）
│  └─ SOPS + Age（加密后提交 Git，无平台依赖）
├─ 场景：企业级动态凭据、字段级访问
│  └─ Vault Agent Sidecar（最安全，但最复杂）
└─ 黄金法则：无论选哪种方案，密钥永不明文提交到 Git！
```

## GitOps 最佳实践

### 仓库结构建议

```
gitops-infra/
├── clusters/              # 集群级别的配置
│   └── prod-cluster/
│       ├── cert-manager/
│       ├── ingress-nginx/
│       └── monitoring/
├── apps/                  # 应用配置
│   ├── payment-service/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   └── web-frontend/
│       ├── dev/
│       └── prod/
├── projects/              # ArgoCD Project 定义
│   └── team-a-project.yaml
└── argocd/                # ArgoCD 自身配置
    └── argocd-cm.yaml
```

### 安全与隔离

1. **环境隔离**：不同环境的仓库路径/分支，有不同权限
2. **签名验证**：使用 GPG 签名 commit，ArgoCD 验证签名
3. **密钥管理**：Git 中不放明文密钥，通过 SealedSecret 或 External Secrets Operator
4. **PR 流程**：所有环境变更走 PR，完成 Code Review

### 常见反模式

| 反模式 | 问题 | 改进 |
|--------|------|------|
| 直接 git push 到 main | 绕过评审 | 强制 PR + 分支保护 |
| 手动 kubectl edit 线上 | 产生漂移 | ArgoCD selfHeal 自动修正 |
| 密钥明文存 Git | 泄密风险 | SealedSecret / Vault |
| 一个仓库管理所有 | 权限难隔离 | 按环境/团队拆分 |
| 将 CI 产物（镜像 Tag）写死在 YAML | 无法自动化升级 | ArgoCD Image Updater |

## 生产案例：某金融科技公司用 GitOps 统一管理 200+ K8s 集群

### 背景

某金融科技公司，管理超过 **200 个 K8s 集群**，分布在 **6 个 Region**（美东、美西、欧洲、新加坡、日本、澳洲），每个 Region 包含生产、预发布、测试环境。服务全球 5000 万+ 用户，高峰期每秒处理数万笔交易。

### 痛点

| 问题 | 影响 |
|------|------|
| **传统 CI/CD 直连集群** | Jenkins/Spinnaker 直连每个集群的 API Server，集群凭证（kubeconfig）分散在多个 CI 系统中，安全审计困难 |
| **新集群接入慢** | 每新增一个集群，需要手动配置 CI 连接、配置 CD 流水线、配置监控告警，平均需要 3 天 |
| **环境配置漂移** | 不同集群的 Ingress、证书、日志配置不一致，排障困难 |
| **部署一致性差** | 同一应用在不同 Region 的配置存在细微差异，导致"在新加坡跑不起来"的问题 |
| **安全审计黑盒** | 谁在什么时间修改了什么配置？没有统一的审计日志 |

### 解决方案：ArgoCD + ApplicationSet

#### 架构设计

```
GitOps 控制平面（部署在管理集群）
┌──────────────────────────────────────────┐
│  ArgoCD Control Plane                      │
│  ┌────────────────────────────────────┐   │
│  │  ApplicationSet Controller          │   │
│  │  ┌──────────────────────────────┐ │   │
│  │  │  Cluster Generator          │ │   │
│  │  │  （自动发现集群）             │ │   │
│  │  └──────────────────────────────┘ │   │
│  │  ┌──────────────────────────────┐ │   │
│  │  │  Git Generator              │ │   │
│  │  │  （按目录结构生成 App）        │ │   │
│  │  └──────────────────────────────┘ │   │
│  └────────────────────────────────────┘   │
└──────────────────┬───────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
┌─────────┐ ┌─────────┐   ┌─────────┐
│ Region1 │ │ Region2 │...│ RegionN │ (200+ 集群)
│ 生产    │ │ 预发布   │   │ 测试    │
│ 100个   │ │ 60个    │   │ 40+个   │
└─────────┘ └─────────┘   └─────────┘
```

#### 关键实施细节

**1. Git 仓库分层**

```
gitops-infra/
├── clusters/                    # 集群注册
│   ├── us-east-1/
│   │   ├── prod/                # 集群配置（API Server URL, CA Cert）
│   │   ├── staging/
│   │   └── test/
│   ├── eu-west-1/
│   │   ├── prod/
│   │   └── staging/
│   └── ap-southeast-1/
│       ├── prod/
│       └── test/
├── applications/               # 应用部署配置
│   ├── payment-service/
│   │   ├── base/               # 公共基配置
│   │   ├── overlays/
│   │   │   ├── prod/           # 生产环境覆写（高配置、多副本）
│   │   │   ├── staging/        # 预发布配置
│   │   │   └── test/           # 测试环境配置
│   │   └── kustomization.yaml
│   ├── user-service/
│   └── trading-engine/
├── infrastructure/             # 基础设施配置
│   ├── ingress-nginx/
│   ├── cert-manager/
│   ├── monitoring/
│   └── logging/
└── projects/                   # ArgoCD Project RBAC
    ├── team-core.yaml            # 核心业务团队
    ├── team-infrastructure.yaml   # 基础设施团队
    └── team-data.yaml             # 数据平台团队
```

**2. ApplicationSet 自动管理集群**

```yaml
# ApplicationSet：按集群生成 Application
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-clusters
  namespace: argocd
spec:
  generators:
  - clusters:                        # 自动发现所有已注册集群
      selector:
        matchLabels:
          environment: prod
          compliance: pci-dss       # PCI-DSS 合规集群
  - git:                             # 按仓库目录结构生成
      repoURL: https://git.internal/team/gitops-infra.git
      revision: main
      directories:
      - path: applications/*/overlays/prod
  template:
    metadata:
      name: '{{name}}-{{path.basenameNormalized}}'
    spec:
      project: 'team-core'
      source:
        repoURL: https://git.internal/team/gitops-infra.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: '{{server}}'
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

**3. Sync Waves 控制资源创建顺序**

```yaml
# 使用 Sync Waves 确保资源按依赖顺序创建
# Wave 0: CRDs 和命名空间
# Wave 1: 配置和密钥
# Wave 2: 存储和数据层
# Wave 3: 核心业务服务
# Wave 4: 网关和 Ingress

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "2"   # 存储层
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  annotations:
    argocd.argoproj.io/sync-wave: "3"   # 核心服务
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: external-gateway
  annotations:
    argocd.argoproj.io/sync-wave: "4"   # 网关层
```

**4. ArgoCD RBAC 权限划分**

```yaml
# argocd-rbac-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.rbac.v2: |
    # 基础设施团队——全集群基础设施组件管理
    g, team-infrastructure, role:infra-admin

    # 核心业务团队——生产 Core 项目读写
    g, team-core, role:core-developer

    # 数据团队——数据平台项目只读
    g, team-data, role:data-viewer

    # 只读审计
    g, auditors, role:readonly-all

    # 自定义角色
    p, role:infra-admin, projects, get, *, allow
    p, role:infra-admin, applications, *, *, allow
    p, role:infra-admin, clusters, *, *, allow

    p, role:core-developer, projects, get, prod-core, allow
    p, role:core-developer, applications, *, prod-core/*, allow

    p, role:data-viewer, projects, get, data-platform, allow
    p, role:data-viewer, applications, get, data-platform/*, allow
```

### 实施效果

| 关键指标 | 实施前 | 实施后 | 提升幅度 |
|---------|-------|-------|---------|
| **新集群接入时间** | 3 天（手动配置 CI/CD/监控） | 15 分钟（注册集群信息到 Git） | **96% 提升** |
| **部署一致性** | 存在配置漂移（不同 Region 差异） | 100%（所有集群从相同 Git 仓库拉取） | **完全一致** |
| **安全审计** | 无统一审计日志 | 每个变更对应一个 Git Commit | **零审计问题** |
| **回滚时间** | 15-30 分钟（手动执行 rollout） | 1 分钟（`git revert`） | **93% 提升** |
| **部署频率** | 每周 1-2 次（审批周期长） | 每日多次（全自动化流水线） | **400%+ 提升** |
| **人力占用** | 5 人运维团队全职处理集群配置 | 1 人半职（维护 GitOps 仓库模板） | **90% 人力节省** |
| **安全事件** | 每年 2-3 次凭证泄露/误操作 | 零 | **100% 改善** |

### 关键经验总结

```
├─ 采用 ApplicationSet 是 200+ 集群管理的核心杠杆
│  一次模板定义，自动适配所有集群
├─ Sync Waves 是避免"先部署应用后创建数据库"的关键
│  没有 Sync Waves，200 个集群同时出错后果严重
├─ RBAC 权限必须与团队组织架构对齐
│  每个项目（Project）映射一个业务团队
├─ 不要直接给所有人 Git 写权限
│  使用 CODEOWNERS + PR 审批
└─ 密钥管理先于一切
   外部 Secrets Operator（Vault）统一管理所有集群的密钥
```

## 本章小结

| 要点 | 说明 |
|------|------|
| GitOps 核心 | Git 为唯一真相源，Agent 确保集群状态与 Git 一致 |
| Pull 模式 | 集群 Agent 主动从 Git 拉取，比传统 Push 更安全 |
| ArgoCD vs Flux | 大集群选 ArgoCD（多集群+企业级），小团队选 Flux（简洁） |
| CI 与 GitOps | CI 仅负责构建镜像和更新 manifests，GitOps 负责同步 |
| Tag 策略 | 推荐 Commit SHA 或 SemVer+SHA，确保 Tag 不可变 |
| 多环境管理 | Kustomize base/overlay 或 Helm values 实现环境差异化 |
| 非 K8s GitOps | Terraform Cloud / Crossplane / Ansible AWX 均可 GitOps 化 |
| 密钥管理 | SealedSecrets / External Secrets Operator / SOPS，坚决不明文 |
| 安全基线 | 环境隔离、签名验证、密钥不落 Git |
| 回滚即 `git revert` | 一次回滚 = 将 Git 恢复到上次已知良好状态 |

