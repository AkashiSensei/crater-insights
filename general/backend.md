# 后端项目结构

本文档记录了项目后端的架构设计和关键机制。

## 更新记录

> **2025-12-04** | [chore: fix backend workflow cache (#302)](https://github.com/raids-lab/crater/commit/01ba569b267e8c25c489edbd66648c6ab79842f2) | `01ba569`  
> 修复了 GitHub Actions workflow 中的 Go 依赖缓存配置，在 `actions/setup-go@v6` 中添加了 `cache-dependency-path` 参数，确保正确缓存 Go 模块依赖。

## 路由注册机制

后端采用基于 Manager 接口的路由注册机制，每个业务模块实现 `Manager` 接口，通过 `RegisterPublic`、`RegisterProtected`、`RegisterAdmin` 三个方法注册不同类型的路由。

### 路由前缀常量

定义在 `pkg/constants/const.go`：

```go
APIPrefix        = "api"           // 公开接口前缀（无需登录）
APIV1Prefix      = "api/v1"       // 用户接口前缀（需要登录）
APIV1AdminPrefix = "api/v1/admin" // 管理员接口前缀（需要登录+平台管理员权限）
```

### 路由注册流程

路由注册在 `internal/route.go` 的 `RegisterService` 方法中进行，支持三种类型的路由：

**1. Public 路由**（无需登录）
- **前缀**：`api/{manager_name}`
- **中间件**：无
- **注册方法**：`mgr.RegisterPublic(publicRouter.Group(mgr.GetName()))`
- **示例**：`/api/auth/login`

**2. Protected 路由**（需要登录）
- **前缀**：`api/v1/{manager_name}`
- **中间件**：`AuthProtected()`（自动验证 JWT token）
- **注册方法**：`mgr.RegisterProtected(protectedRouter.Group(mgr.GetName()))`
- **示例**：`/api/v1/accounts`
- **注意**：注册后会自动验证登录状态，函数体内无需再验证 token。但**账户管理员权限**（`RoleAccount == RoleAdmin`）需要在函数体内手动检查。

**3. Admin 路由**（需要登录+平台管理员权限）
- **前缀**：`api/v1/admin/{manager_name}`
- **中间件**：`AuthProtected()` + `AuthAdmin()`（自动验证登录和平台管理员权限）
- **注册方法**：`mgr.RegisterAdmin(adminRouter.Group(mgr.GetName()))`
- **示例**：`/api/v1/admin/accounts`
- **注意**：注册后会自动验证登录状态和平台管理员权限（`RolePlatform == RoleAdmin`），函数体内无需再验证。

### Manager 接口

每个业务模块需要实现以下接口（定义在 `internal/handler/interface.go`）：

```go
type Manager interface {
    GetName() string
    RegisterPublic(group *gin.RouterGroup)
    RegisterProtected(group *gin.RouterGroup)
    RegisterAdmin(group *gin.RouterGroup)
}
```

**实现要求**：
- **必须实现所有方法**：Go 接口要求实现所有声明的方法，否则编译不通过
- **可以为空实现**：如果某个类型的路由不需要，可以提供空实现
- **示例**：`func (mgr *XxxMgr) RegisterPublic(_ *gin.RouterGroup) {}`（下划线 `_` 表示不使用参数）

### 接口注册策略

**场景**：接口用户和管理员都可以使用，但未登录不能使用

#### 推荐方式：共享处理函数

在 `RegisterProtected` 和 `RegisterAdmin` 中都注册，调用同一个处理函数。

- **注册位置**：
  - `RegisterProtected`（生成 `/api/v1/{manager_name}/...`）
  - `RegisterAdmin`（生成 `/api/v1/admin/{manager_name}/...`）
- **处理函数**：同一个函数，内部通过 `util.GetToken(c)` 判断身份执行不同逻辑
- **优点**：
  - 管理员使用带 admin 前缀的 URL，符合 RESTful 规范，权限清晰
  - 代码不重复，同一个函数处理两种场景
- **注意**：会产生两个不同的路径，前端需要根据用户身份选择调用哪个

#### 方式2：分离处理函数

在 `RegisterProtected` 和 `RegisterAdmin` 中都注册，调用不同的处理函数。

- **注册位置**：
  - `RegisterProtected`（生成 `/api/v1/{manager_name}/...`）
  - `RegisterAdmin`（生成 `/api/v1/admin/{manager_name}/...`）
- **处理函数**：分别为用户版本和管理员版本的两个不同函数
- **优点**：逻辑分离清晰，职责明确
- **常见做法**：如果用户和管理员有部分相同逻辑，可以抽象成公共函数，然后在各自的处理函数中调用
- **适用场景**：需要明确区分用户和管理员处理流程的场景（即使有部分相同逻辑）

#### 方式3：仅注册 Protected（禁止使用）

只注册在 `RegisterProtected`，管理员也使用用户接口。

- **注册位置**：仅在 `RegisterProtected`（生成 `/api/v1/{manager_name}/...`）
- **禁止原因**：
  - 管理员使用用户接口 URL，不符合 RESTful 规范
  - URL 无法体现权限层级，管理员和用户使用相同的 URL
  - 违反了"管理员应使用 admin 前缀接口"的设计原则

#### 选择建议

- **逻辑基本相同，仅数据范围不同** → 使用**推荐方式**（两个地方注册，调用同一个函数）
- **逻辑完全不同** → 使用**方式2**（两个地方注册，调用不同的函数）
- **禁止使用方式3**：管理员必须使用带 `admin` 前缀的接口

