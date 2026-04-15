# Crater 付费功能锁（Feature Lock）实现方案调研报告

> 调研日期：2026-04-14  
> 调研目标：为 Crater（Go 后端 + TypeScript 前端 DevOps 平台）提供完整源码交付前提下的付费功能锁实现方案

---

## 目录

1. [背景与核心问题](#1-背景与核心问题)
2. [业界主流方案概览](#2-业界主流方案概览)
3. [方案一：纯源码开放（GitLab 模式）](#3-方案一纯源码开放gitlab模式)
4. [方案二：Feature Flag 功能开关](#4-方案二feature-flag-功能开关)
5. [方案三：离线 License Key 验证](#5-方案三离线-license-key-验证)
6. [方案四：在线 License 验证](#6-方案四在线-license-验证)
7. [方案五：代码混淆与模块化锁定](#7-方案五代码混淆与模块化锁定)
8. [商业 License 服务调研](#8-商业-license-服务调研)
9. [Go 语言 License 验证相关库](#9-go-语言-license-验证相关库)
10. [Crater 代码结构分析](#10-crater-代码结构分析)
11. [各方案对比](#11-各方案对比)
12. [对 Crater 的具体建议](#12-对-crater-的具体建议)

---

## 1. 背景与核心问题

### 1.1 核心问题

用户项目 Crater 是一个**自托管 DevOps 平台**（类 GitLab 方案），技术栈为：
- **后端**：Go 语言（Gin 框架 + GORM + Kubernetes Controller）
- **前端**：TypeScript（TanStack Router + React 类框架）

用户希望：**在完整交付全部源码的基础上**，实现部分高级功能不付费就不能用。

这与 GitLab 的做法不同——GitLab 的方式是将部分源码完全不开源（EE features 在独立仓库），而不是在源码中加锁。

### 1.2 需要回答的问题

1. 是否可以在完整交付源码的基础上实现功能锁？
2. 如果可以，有哪些成熟的技术方案？
3. 每种方案的优缺点是什么？
4. 结合 Crater 现有代码结构，如何落地？

---

## 2. 业界主流方案概览

经过调研，业界实现付费功能锁的主流方案分为以下几大类：

| 方案 | 代表产品 | 是否交付源码 | 防破解强度 | 实施难度 |
|------|---------|------------|-----------|---------|
| 纯源码开放（部分代码不开源） | GitLab CE/EE | 部分 | ★★★★★ | 高（需维护双仓库） |
| Feature Flag 开关 | 大多数 SaaS 化软件 | 全部 | ★★☆☆☆ | 低 |
| 离线 License Key | JetBrains、HashiCorp | 全部 | ★★★☆☆ | 中 |
| 在线 License 验证 | GitLab EE、SonarQube Enterprise | 全部 | ★★★★☆ | 中高 |
| 代码混淆+模块化锁定 | 传统商业软件 | 全部 | ★★★★☆ | 高 |
| 第三方商业 License 服务 | LicenseSCout/Keygen/Cryptolens | 全部 | ★★★★☆ | 中 |

---

## 3. 方案一：纯源码开放（GitLab 模式）

### 3.1 GitLab 的实现方式

GitLab 采用的是**开放核心（Open Core）**模式：

- **GitLab CE（社区版）**：在 MIT 许可证下开源，代码托管在 `gitlabhq/gitlabhq` 仓库
- **GitLab EE（企业版）**：不在公共仓库公开，额外功能代码在 `ee` 目录
- **GitLab FOSS**：专门维护了一个去除了专有代码的开源版本 `gitlab-org/gitlab-foss`

**核心机制**：
- EE 的额外功能代码完全不存在于 CE 仓库中
- 两套代码共享同一个 CI/CD 框架，但 EE 目录下的代码不对外开放
- GitLab 内部使用 Feature Flag 系统（Flipper）来管理功能灰度发布，但 EE 的区分主要靠代码隔离而非运行时验证

**参考链接**：
- GitLab CE 镜像仓库：https://github.com/gitlabhq/gitlabhq
- GitLab FOSS 仓库：https://gitlab.com/gitlab-org/gitlab-foss/
- GitLab 官方定价页面：https://about.gitlab.com/pricing/
- GitLab 架构文档：https://docs.gitlab.com/ee/development/architecture.html

### 3.2 对 Crater 的适用性

**优点**：
- 防破解性最强（代码根本不存在，无法绕过）
- 法律保护最有力

**缺点**：
- **用户要求完整交付源码**，这与 GitLab 模式根本冲突
- 需要维护多个代码仓库，分支管理复杂
- 社区贡献难以同时惠及 CE 和 EE

**结论**：与用户需求（完整交付源码）相悖，**不推荐**。

---

## 4. 方案二：Feature Flag 功能开关

### 4.1 原理

在代码中嵌入功能开关（Feature Flag），通过配置文件或环境变量控制功能开启/关闭。

```
// 示例代码
if featureFlags.IsEnabled("advanced_analytics") {
    // 执行高级分析功能
} else {
    // 返回"请升级"提示
}
```

### 4.2 业界使用情况

几乎所有现代 SaaS 软件都使用 Feature Flag 进行功能灰度发布，但作为**付费功能锁**的强度较低，因为：
- 配置以明文存储在文件中
- 用户可以通过修改配置文件绕过

**Go 生态中的 Feature Flag 库**：
- `github.com/thomasjpfan/go-feature-flag` - 支持多种后端（文件、Redis、LaunchDarkly 等）
- `github.com/nicholaspcr/gate` - 轻量级 Go 功能开关
- `unleash/go-proxy` - Unleash 的 Go SDK

**参考链接**：
- go-feature-flag 官方文档：https://thomasjpfan.github.io/go-feature-flag/
- Unleash 官网：https://www.getunleash.io/
- LaunchDarkly Feature Flag 平台：https://launchdarkly.com/

### 4.3 对 Crater 的适用性

**优点**：
- 实施成本低，与 Crater 现有架构契合（已有 `pkg/constants` 等配置）
- 前端和后端都可以轻松集成
- 支持灰度发布，不仅用于付费锁定

**缺点**：
- 防破解强度低，有技术能力的用户可以通过修改配置绕过
- 需要用户手动配置，适合有一定技术背景的用户

**结论**：适合作为**辅助方案**，但不建议作为主要的防破解机制。

---

## 5. 方案三：离线 License Key 验证

### 5.1 原理

使用非对称加密（如 RSA 或 Ed25519）对 License 文件进行签名，程序在启动时验证签名有效性。

**典型流程**：
1. 软件开发商使用私钥对 `{feature_set, expiry_date, customer_id}` 数据进行签名
2. 将签名后的 License 文件（JSON 或自定义格式）发给客户
3. 软件在启动/运行时使用内置公钥验证 License 文件签名
4. 验证通过则解锁对应功能

**Go 实现示例**（使用 Ed25519）：
```go
import "filippo.io/edwards25519"

// 验证签名
func ValidateLicense(licenseData []byte, signature []byte, publicKey []byte) bool {
    point, _ := new(edwards25519.Point).SetBytes(publicKey)
    sig, _ := new(edwards25519.Signature).SetBytes(signature)
    msg := hash.Message([]byte(licenseData))
    return point.Verify(&sig, msg)
}
```

**注**：Crater 的 `go.mod` 中已经包含 `filippo.io/edwards25519`（作为 indirect 依赖）。

### 5.2 业界使用情况

- **JetBrains**（IntelliJ IDEA 等）：使用离线 License Server，License 文件包含加密签名
- **HashiCorp**（Terraform, Vault 等）：从 BSL 1.1 转型时使用 License 文件机制，2023 年开始要求连接 HashiCorp 授权服务器验证 License

**HashiCorp License 参考**：
- HashiCorp License 官方文档：https://www.hashicorp.com/products/vagrant/source-license
- HashiCorp License 实现（GitHub）：https://github.com/hashicorp

### 5.3 对 Crater 的适用性

**优点**：
- 不需要连接外部服务器，适合离线/内网环境
- 源码全部交付，客户拥有完整控制权
- 防篡改（签名验证防止伪造 License）
- Ed25519/RSA 签名技术成熟可靠

**缺点**：
- 如果私钥泄露，任何人都可以生成有效 License（需要保护好私钥）
- 每次发布新功能需要更新 License 格式或重新颁发 License
- 无法吊销已颁发的 License（除非内置吊销列表）

**结论**：**强烈推荐**，是"完整交付源码 + 功能锁"场景下最平衡的方案。

---

## 6. 方案四：在线 License 验证

### 6.1 原理

软件在启动或定期运行时，通过网络连接到授权服务器，验证 License 是否有效。

**典型流程**：
1. 软件携带 License Key 向授权服务器发起验证请求
2. 服务器返回验证结果（有效/过期/被吊销）及对应的功能列表
3. 软件根据返回结果决定是否解锁功能

### 6.2 GitLab EE 的实现方式

GitLab EE 通过在线验证来判断用户是否有有效的订阅：
- 免费版用户无法使用 EE 专属功能
- GitLab 会检查许可证是否过期

**参考链接**：
- GitLab License 管理文档：https://docs.gitlab.com/ee/user/admin_area/license.html

### 6.3 对 Crater 的适用性

**优点**：
- 可实时吊销过期或被盗的 License
- 可远程更新功能列表（如新增付费功能）
- 可以统计用户使用情况

**缺点**：
- **需要连接外部服务器**，不适用于完全内网/离线环境
- 增加系统复杂度和运维成本（需要维护授权服务器）
- 如果授权服务器宕机，可能影响用户使用

**结论**：适合有条件部署配套授权服务器的团队，不适合纯内网部署场景。

---

## 7. 方案五：代码混淆与模块化锁定

### 7.1 代码混淆

对 Go 二进制进行混淆（LLVM-obfuscator、Garble 等），使付费功能的代码逻辑难以被逆向分析。

**Go 代码混淆工具**：
- `github.com/burrowers/garble` - Go 代码混淆工具，可混淆函数名、字符串常量等

**参考链接**：
- Garble GitHub：https://github.com/burrowers/garble

### 7.2 模块化锁定

将付费功能编译为独立插件（Go Plugin 系统），只有有效 License 才加载对应插件。

```go
// 仅在 License 验证通过后加载付费模块
plugin, err := plugin.Open("premium_module.so")
if err == nil {
    // 执行付费功能
}
```

**注意**：Go Plugin 在 macOS 上支持有限，且需要 CGO，部署复杂度较高。

### 7.3 对 Crater 的适用性

**优点**：
- 代码混淆增加逆向难度
- 插件化可以将付费代码以二进制形式分发（源码不泄露）

**缺点**：
- Go Plugin 系统不稳定（文档中标注为 EXPERIMENTAL）
- 混淆后的代码难以调试
- 不符合"完整交付源码"的需求

**结论**：不推荐。代码混淆会降低可维护性，模块化会破坏源码完整性。

---

## 8. 商业 License 服务调研

### 8.1 Keygen

**官网**：https://keygen.sh/

Keygen 是一个面向开发者的软件授权和分发 API，支持：
- License Key 生成、激活、吊销
- 机器级别的激活限制
- 离线验证（通过 License 文件）
- 在线验证
- 自托管（Keygen EE）或使用 Keygen Cloud

**特点**：
- 专为开发者设计，SDK 支持 Go、Python、Ruby、Node.js 等
- 支持离线 License 文件验证
- 有免费的自托管版本（Keygen CE）

**Keygen CE GitHub**：https://github.com/keygen-sh/keygen-api

**Go SDK 示例**：
```go
license, err := keygen.Validate(fingerprint)
if err == keygen.ErrLicenseExpired {
    // License 已过期
}
```

**参考链接**：
- Keygen 官网：https://keygen.sh/
- Keygen API 文档：https://keygen.sh/docs/
- Keygen Go SDK：https://github.com/keygen-sh/keygen-go

### 8.2 Cryptolens (Devolens)

**官网**：https://cryptolens.io/

Cryptolens 是一个老牌软件授权平台，近年品牌升级为 Devolens：
- 支持离线许可证验证（使用非对称加密）
- 支持在线许可证验证
- 提供丰富的仪表盘管理 License

**特点**：
- 离线许可证基于 RSA-2048 签名
- 有 Docker 自托管选项
- 商业使用需要付费

**参考链接**：
- Cryptolens/Devolens 官网：https://cryptolens.io/
- Cryptolens GitHub：https://github.com/Cryptolens

### 8.3 LicenseSCout

**GitHub**：https://github.com/LSportsScout/licensescout

LicenseSCout 是一个开源的 License 检测工具，主要用于：
- 检测项目依赖的 License 合规性
- 不是功能锁工具，而是 License 合规管理工具

**参考链接**：
- LicenseSCout GitHub：https://github.com/LSportsScout/licensescout

### 8.4 其他开源方案

| 名称 | 类型 | 特点 |
|------|------|------|
| License Key Manager | 开源 | 简单的 License 生成和验证工具 |
| Simple License System | 开源 | 基于 PHP 的 License 管理 |
| Hammock | 商业 | 软件授权和分发平台 |

---

## 9. Go 语言 License 验证相关库

### 9.1 密码学基础库

| 库 | 用途 | 状态 |
|----|------|------|
| `filippo.io/edwards25519` | Ed25519 签名（已用于 Crater） | 稳定 |
| `golang.org/x/crypto` | RSA、AES、bcrypt 等加密原语 | 官方维护 |
| `github.com/google/tink` | Google 加密库 | 稳定 |

### 9.2 Feature Flag 库

| 库 | 特点 | 链接 |
|----|------|------|
| `thomasjpfan/go-feature-flag` | 支持多种后端、简单易用 | https://github.com/thomasjpfan/go-feature-flag |
| `nicholaspcr/gate` | 轻量级功能开关 | https://github.com/nicholaspcr/gate |
| `bitfield/feature` | 简单的功能开关 | https://github.com/bitfield/feature |

### 9.3 License 验证相关

| 库 | 特点 | 链接 |
|----|------|------|
| `mcuadros/go-license-detector` | 检测项目 License 类型（不是功能锁） | https://github.com/mcuadros/go-license-detector |
| `keygen-sh/keygen-go` | Keygen 官方 Go SDK | https://github.com/keygen-sh/keygen-go |
| `burrowers/garble` | Go 代码混淆 | https://github.com/burrowers/garble |

---

## 10. Crater 代码结构分析

### 10.1 后端结构

```
crater/backend/
├── cmd/crater/main.go           # 入口，版本信息通过 ldflags 注入
├── internal/
│   ├── route.go                 # 路由注册（Public/Protected/Admin 三层）
│   ├── handler/                  # 各功能模块的 HTTP Handler
│   │   ├── interface.go          # Manager 接口定义（GetName, RegisterPublic/Protected/Admin）
│   │   ├── aijob/                # AI 任务模块
│   │   ├── vcjob/               # VC 任务模块
│   │   ├── gpu_analysis.go      # GPU 分析模块
│   │   ├── approvalorder.go     # 审批模块
│   │   ├── metrics.go            # 指标模块
│   │   ├── statistics.go         # 统计模块
│   │   └── ...
│   ├── middleware/
│   │   └── jwt.go               # JWT 认证中间件
│   ├── service/
│   │   ├── config_service.go
│   │   └── gpu_analysis_service.go
│   └── ...
├── pkg/
│   ├── constants/               # 常量定义
│   ├── apis/                    # Kubernetes CRD API 定义
│   ├── crclient/                # CRD 客户端封装
│   └── ...
└── go.mod
```

**关键发现**：
1. Crater 使用**三层权限模型**：Public（无需登录）→ Protected（需登录）→ Admin（需管理员角色）
2. 每个 Handler 模块都实现了 `Manager` 接口，独立注册自己的路由
3. JWT 认证已内置，基于 `github.com/golang-jwt/jwt/v5`
4. 版本信息通过 `ldflags` 在构建时注入（`AppVersion`, `CommitSHA`, `BuildType` 等）

### 10.2 前端结构

```
crater/frontend/src/
├── router.tsx                   # TanStack Router 配置
├── routeTree.gen.ts            # 自动生成的路由树
├── routes/
│   ├── admin/                  # 管理后台路由（accounts, cluster, jobs, users...）
│   ├── auth/                   # 认证路由
│   ├── ingress/
│   ├── portal/
│   └── index.tsx
├── services/
│   ├── api/                    # API 调用封装
│   ├── client.ts
│   └── types.ts
├── hooks/                      # React Hooks
├── components/                 # 共享组件
└── utils/
    └── store/                  # 状态管理
```

**关键发现**：
1. 前端路由使用 TanStack Router（React Router v7）
2. 有独立的管理后台路由（`/admin/*`）
3. 有 API 服务层（`services/api/`）

### 10.3 现有 Feature 相关代码

经过扫描，Crater 代码库中**暂无任何 License 或 Feature Flag 机制**。目前只有 Apache License 头注释和普通的功能模块目录。

### 10.4 实施功能锁的代码切入点

| 层次 | 切入点 | 说明 |
|------|--------|------|
| **后端中间件层** | `middleware/jwt.go` | 在 JWT 认证后插入 License 验证逻辑 |
| **后端 Handler 层** | 各 `Manager.RegisterProtected()` | 为特定路由添加付费验证 |
| **后端服务层** | `service/config_service.go` | 新建 License 服务 |
| **后端启动层** | `cmd/crater/main.go` | 启动时验证 License |
| **前端路由层** | `routes/admin/*.tsx` | 隐藏或禁用未付费功能入口 |
| **前端 API 层** | `services/api/` | 在 API 响应中附加功能可用性信息 |

---

## 11. 各方案对比

### 11.1 综合对比表

| 维度 | 纯源码开放 | Feature Flag | 离线 License Key | 在线 License 验证 | 代码混淆+模块化 |
|------|-----------|-------------|-----------------|-----------------|----------------|
| **完整交付源码** | ❌ | ✅ | ✅ | ✅ | ⚠️ 部分 |
| **防破解强度** | ★★★★★ | ★★☆☆☆ | ★★★☆☆ | ★★★★☆ | ★★★★☆ |
| **实施难度** | 高 | 低 | 中 | 中高 | 高 |
| **内网/离线可用** | - | ✅ | ✅ | ❌ | ✅ |
| **可吊销 License** | - | ❌ | ❌ | ✅ | ❌ |
| **无外部依赖** | - | ✅ | ✅ | ❌ | ✅ |
| **维护成本** | 高（双仓库） | 低 | 低 | 高 | 高 |
| **业界成熟度** | 高 | 高 | 高 | 高 | 低（Go 生态） |

### 11.2 方案选择决策树

```
是否需要完整交付源码？
├── 否 → 选择 GitLab 模式（维护 CE/EE 双仓库）
└── 是 → 是否有外网连接需求？
    ├── 否 → 选择离线 License Key（Ed25519/RSA 签名）
    └── 是 → 是否需要远程吊销功能？
        ├── 是 → 选择在线 License 验证（自建或 Keygen Cloud）
        └── 否 → 选择离线 License Key + 定期重新签发
```

---

## 12. 对 Crater 的具体建议

### 12.1 推荐方案：离线 License Key + Feature Flag 双层架构

**核心思路**：采用"离线签名验证"作为主要防破解机制，配合"Feature Flag"作为灵活的付费功能管理框架。

**分层设计**：

#### 第一层：License 验证（防破解）
- 使用 **Ed25519** 签名验证（Go 标准库已有支持 `filippo.io/edwards25519`，已作为间接依赖引入）
- License 文件格式示例：
  ```json
  {
    "customer_id": "CUST-001",
    "features": ["gpu_analysis", "approval_workflow", "enterprise_sso"],
    "max_users": 100,
    "expiry_date": "2027-12-31",
    "issued_at": "2025-01-01"
  }
  ```
- 对上述 JSON 内容计算 SHA-256 后用 Ed25519 签名，签名附在 License 文件中
- Crater 内置公钥，验证签名有效性后解析 features 列表

#### 第二层：Feature Flag（功能管理）
- 建立统一的 Feature Flag 配置（可在数据库或配置文件中存储）
- 每个付费功能对应一个 Feature Flag Key
- Handler 层检查 Flag 状态，未付费返回友好提示而非直接拒绝

### 12.2 具体实施步骤

#### Step 1：新建 License Service（后端）

```
pkg/license/
├── license.go         # License 结构体定义和验证逻辑
├── verifier.go        # Ed25519 签名验证
├── features.go        # Feature Flag 定义
└── errors.go          # 自定义错误类型
```

**核心接口**：
```go
type LicenseVerifier interface {
    LoadLicense(path string) (*License, error)
    Validate(l *License) error  // 验证签名、过期时间等
    IsFeatureEnabled(l *License, feature string) bool
}
```

#### Step 2：中间件集成

在 `middleware/` 下新建 `license.go`，在 Protected 路由层注入 License 验证：

```go
// 验证用户的 License 状态
func LicenseVerified() gin.HandlerFunc {
    return func(c *gin.Context) {
        license := getSystemLicense() // 从内存/缓存获取
        if license == nil || license.Expired() {
            c.JSON(402, gin.H{"error": "license_invalid", "message": "请激活您的 License"})
            c.Abort()
            return
        }
        c.Set("license", license)
        c.Next()
    }
}
```

#### Step 3：Handler 层接入

在需要付费验证的 Handler 中：

```go
func (m *gpuAnalysisManager) RegisterProtected(group *gin.RouterGroup) {
    // 免费功能：直接注册
    group.GET("/basic", m.basicStats)
    
    // 付费功能：先检查 License
   付费Group := group.Group("/advanced")
    付费Group.Use(middleware.LicenseProtected("gpu_analysis"))
    付费Group.GET("/detailed", m.advancedStats)
}
```

#### Step 4：前端适配

在 `services/api/` 下新建 License 相关 API 调用：
- `GET /api/v1/license/status` - 获取当前 License 状态和可用功能
- 在路由守卫（TanStack Router）中检查 License 状态
- 未付费的功能入口显示"升级"按钮而非 404

**前端文件建议**：
```
src/services/api/license.ts    # License API 调用
src/hooks/useLicense.ts       # License 状态 Hook
src/routes/admin/gpu-analysis/ # 付费功能路由守卫
```

#### Step 5：Admin 管理界面

在 `routes/admin/` 下新增 License 管理页面：
- 上传 License 文件
- 查看当前 License 状态
- 查看可用功能列表

### 12.3 License 生成工具（辅助）

需要为 Crater 开发一个配套的 **License 生成工具**（CLI），供销售/运营使用：

```
crater-license-tool generate \
  --customer-id "CUST-001" \
  --features "gpu_analysis,approval_workflow" \
  --max-users 100 \
  --expiry 2027-12-31 \
  --private-key /path/to/private_key.pem \
  --output customer-001.license
```

该工具不在交付范围内，仅供内部使用。

### 12.4 防破解注意事项

即使使用离线签名验证，以下措施可以进一步提高安全性：

1. **代码混淆**：使用 `garble` 对 `pkg/license/` 目录进行混淆，增加逆向难度
2. **反调试**：检测是否在调试模式运行（`os.LookupEnv("DEBUG")`），是则拒绝验证
3. **签名+盐值**：在 License 文件中加入机器特征（如 CPU 序列号、MAC 地址）的哈希，防止 License 文件复制到其他机器
4. **定期重新签发**：设置较短的 License 有效期（如 1 年），到期后需要重新获取新 License
5. **错误信息模糊化**：验证失败时返回通用错误，不要直接暴露签名验证失败的细节

### 12.5 推荐的付费功能分层策略

| 功能层级 | 示例功能 | 建议策略 |
|---------|---------|---------|
| **基础层（免费）** | 基础任务提交、用户管理、基础监控 | 开源版本即可使用 |
| **高级层（付费）** | GPU 深度分析、高级审批流、SSO | License Key 验证 |
| **企业层（高价）** | 多集群管理、审计日志、合规报告 | License Key + 额外限制 |

---

## 参考资料

### GitLab 相关
- GitLab CE 镜像仓库：https://github.com/gitlabhq/gitlabhq
- GitLab FOSS 仓库：https://gitlab.com/gitlab-org/gitlab-foss/
- GitLab License 管理文档：https://docs.gitlab.com/ee/user/admin_area/license.html
- GitLab 架构文档：https://docs.gitlab.com/ee/development/architecture.html
- GitLab 官方定价：https://about.gitlab.com/pricing/

### 开源 DevOps 工具
- Gitea 官方仓库：https://github.com/go-gitea/gitea
- Jenkins 官网：https://www.jenkins.io/
- SonarQube 官网：https://www.sonarsource.com/products/sonarqube/
- SonarQube 定价：https://www.sonarsource.com/plans-and-pricing/

### 商业 License 服务
- Keygen 官网：https://keygen.sh/
- Keygen API 文档：https://keygen.sh/docs/
- Keygen Go SDK：https://github.com/keygen-sh/keygen-go
- Keygen 自托管版（GitHub）：https://github.com/keygen-sh/keygen-api
- Cryptolens/Devolens 官网：https://cryptolens.io/
- Cryptolens GitHub：https://github.com/Cryptolens
- LicenseSCout GitHub：https://github.com/LSportsScout/licensescout

### Go 相关工具
- go-feature-flag：https://github.com/thomasjpfan/go-feature-flag
- Go 代码混淆工具 Garble：https://github.com/burrowers/garble
- go-license-detector（License 检测，非功能锁）：https://github.com/mcuadros/go-license-detector
- Edwards25519 签名库（已用于 Crater）：https://pkg.go.dev/filippo.io/edwards25519
- golang-jwt：https://github.com/golang-jwt/jwt（Crater 已使用）

### HashiCorp License（参考架构）
- HashiCorp License 文档：https://www.hashicorp.com/products/vagrant/source-license

### Feature Flag 服务
- Unleash 官网：https://www.getunleash.io/
- LaunchDarkly 官网：https://launchdarkly.com/

---

## 总结

**可以做到在完整交付源码的基础上实现付费功能锁**，但需要权衡防破解强度和实施复杂度。

**最优推荐**：离线 Ed25519 License Key 验证 + Feature Flag 双层架构。
- 防破解强度：中等（足够防住大多数非专业用户）
- 内网可用：是
- 实施复杂度：中等
- 外部依赖：无

**补充建议**：
1. 如果预算允许，可以使用 Keygen Cloud（https://keygen.sh/）作为授权服务器，享受专业的 License 管理界面和 API
2. 不要过度依赖防破解手段，保护收入的核心还是产品价值和客户关系
3. 建议先选定 2-3 个付费功能作为试点，验证商业模式后再扩展

---

*报告完成*
