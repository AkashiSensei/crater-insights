# 错误信息返回与展示

描述现在项目前后端错误信息返回的流程以及设计。

## 更新记录

> **2026-01-30** | [feat: dockerfile improve (#337)](https://github.com/raids-lab/crater/commit/b165150856a19db43f61f0a7b45ce7cf3403d88d) | `b165150`  
> 创建错误信息返回与展示文档，系统梳理前后端错误处理机制。文档涵盖后端错误返回设计（响应格式、HTTP 状态码映射、业务错误码定义、响应函数）、前端错误处理流程（错误捕获解析、错误展示格式、错误码同步机制）、错误处理流程总结和已知问题。重点说明当前实现中 HTTP 状态码不符合 RESTful 规范、业务错误码使用不一致、错误消息处理不一致和国际化缺失等问题。

## 错误返回架构概述

Crater 采用**业务错误码 + HTTP 状态码**的双层错误返回机制。业务错误码用于前端进行细粒度的错误处理，HTTP 状态码遵循 RESTful 规范（当前实现不完全符合）。这种设计允许后端返回丰富的业务错误信息，同时保持 HTTP 协议层面的语义。

## 后端错误返回设计

### 响应格式

后端统一使用 `resputil` 包处理 API 响应，所有响应遵循统一的 JSON 格式：

```json
{
  "code": 0,           // 业务错误码（ErrorCode）
  "data": {},          // 响应数据（成功时）或 null（错误时）
  "msg": ""            // 错误消息（错误时）或空字符串（成功时）
}
```

这种统一格式使得前端可以一致地处理所有 API 响应，无论成功还是失败。

### HTTP 状态码映射

**当前实现（不符合 RESTful 规范）：**
- 成功响应：`200 OK`
- 所有错误响应：`500 Internal Server Error`（无论业务错误码是什么）

**注意：** 部分代码使用 `HTTPError` 函数手动指定 HTTP 状态码（如 401、403），但大部分错误处理仍使用 `Error` 函数，统一返回 500。这导致 HTTP 状态码无法准确反映错误的语义类型。

### 业务错误码定义

业务错误码（`ErrorCode`）定义在 `backend/internal/resputil/code.go`，采用分层编码策略，通过错误码的数值范围区分错误类型：

- `0`: 成功（`OK`）
- `40001-40099`: 通用错误（如参数错误、业务逻辑错误）
- `40101-40199`: 认证相关错误（Token 过期、无效凭证等）
- `40301-40399`: 权限相关错误（用户无权限、邮箱未验证等）
- `40401-40499`: 资源未找到
- `50001-50099`: 服务错误
- `99999`: 未指定错误（`NotSpecified`，表示开发者未正确使用错误码）

具体错误码定义请参考 `backend/internal/resputil/code.go`。

分层编码的优势在于可以通过错误码范围快速判断错误类型，但当前实现中大量使用 `NotSpecified (99999)`，削弱了这一设计的价值。

### 后端响应函数

后端提供了四种响应函数，用于不同场景：

1. **`Success(c *gin.Context, data any)`**
   - 成功响应，返回 `200 OK`
   - `code = 0`，`data` 为响应数据

2. **`Error(c *gin.Context, msg string, errorCode ErrorCode)`**
   - 错误响应，返回 `500 Internal Server Error`
   - `code` 为业务错误码，`msg` 为错误消息
   - **问题**：无论业务错误码是什么，都返回 500，不符合 RESTful 规范

3. **`HTTPError(c *gin.Context, httpCode int, err string, errorCode ErrorCode)`**
   - 可手动指定 HTTP 状态码的错误响应
   - 用于需要返回特定 HTTP 状态码的场景（如 401、403）
   - **问题**：需要手动指定，容易不一致

4. **`BadRequestError(c *gin.Context, err string)`**
   - 参数绑定错误，返回 `400 Bad Request`
   - 用于 Gin 参数绑定失败时

## 前端错误处理流程

### 错误捕获与解析

前端使用 `ky` 库进行 HTTP 请求，错误处理分为两个阶段：错误捕获解析和错误展示。

**第一阶段：`apiRequest` 函数**（`frontend/src/services/client.ts`）

`apiRequest` 是前端统一的 API 请求封装，负责捕获和初步处理错误：

1. 捕获 `HTTPError`（来自 `ky` 库）
2. 解析后端返回的 JSON 响应，提取 `{ code, msg }`
3. 将解析后的错误信息挂载到 `error.data` 和 `error.httpStatus`，供后续处理使用
4. 根据业务错误码进行不同处理：
   - `ERROR_INVALID_REQUEST`: 显示 "请求参数有误, {msg}"（使用后端消息，但添加固定前缀）
   - `ERROR_USER_NOT_ALLOWED`: 显示固定消息（**忽略后端返回的消息**）
   - `ERROR_USER_EMAIL_NOT_VERIFIED`: 显示固定消息（**忽略后端返回的消息**）
   - `ERROR_BACKEND`: 显示后端返回的 `msg` 或默认消息
   - `ERROR_NOT_SPECIFIED`: 调用 `showErrorToast(error)` 显示完整错误信息
   - 其他：使用 `toast.error` 显示错误（使用后端消息）

**注意：** `ERROR_USER_NOT_ALLOWED` 和 `ERROR_USER_EMAIL_NOT_VERIFIED` 这两个错误码会完全忽略后端返回的 `msg`，始终显示前端硬编码的固定消息。这意味着后端无法为这些错误码提供自定义的错误提示信息，限制了错误消息的灵活性。

**第二阶段：`showErrorToast` 函数**（`frontend/src/utils/toast.ts`）

`showErrorToast` 是统一的错误展示函数，负责将错误信息格式化为用户可见的 Toast 提示：

- 支持多种错误类型：`AxiosError`（向后兼容）、`HTTPError`（来自 `ky` 库）、`string`、其他错误对象
- 构建统一的错误消息格式：`[HTTP {httpStatus}] [Code {businessCode}] {errorMessage}`

### 前端错误展示格式

前端错误信息展示格式为：

```
[HTTP {httpStatus}] [Code {businessCode}] {errorMessage}
```

例如：
- `[HTTP 500] [Code 99999] regex pattern mismatch: base image 'node:14' does not match required format...`

如果 HTTP 状态码不可用，则只显示：
```
[Code {businessCode}] {errorMessage}
```

这种格式同时展示了 HTTP 协议层面的状态码和业务层面的错误码，有助于调试和问题定位。

### 前端业务错误码同步机制

前端错误码定义在 `frontend/src/services/error_code.ts`，与后端保持一致。

**错误码生成脚本：**

前端错误码通过 `frontend/src/services/generator.py` 脚本从后端错误码定义文件自动生成。脚本解析后端的 Go 错误码定义，生成对应的 TypeScript 常量。

**脚本运行方式：**

脚本需要**手动运行**，当前没有自动化机制（如 pre-commit hooks 或 CI/CD 自动调用）。

运行方式：
- 在 `frontend` 目录下执行：`make generate`
- 或直接运行：`python3 ./src/services/generator.py ../backend/internal/resputil/code.go ./src/services/error_code.ts`

**注意：** 当后端修改了 `backend/internal/resputil/code.go` 中的错误码定义后，需要在前端目录手动执行 `make generate` 来更新前端的错误码文件，以保持前后端错误码同步。这种手动同步机制存在遗漏风险。

**前端错误码使用位置：**

前端错误码主要在以下位置使用：
- `frontend/src/services/client.ts`：`apiRequest` 函数和 Token 刷新逻辑
- 认证相关组件：登录表单、注册表单
- 业务组件：管理员页面、SSH 端口对话框等

## 错误处理流程总结

```
后端 Handler
  ↓
resputil.Error/HTTPError/Success
  ↓
HTTP 响应 { code, data, msg } + HTTP Status Code
  ↓
前端 apiRequest 捕获 HTTPError
  ↓
解析 JSON 响应，挂载到 error.data 和 error.httpStatus
  ↓
根据业务错误码 switch 处理
  ↓
showErrorToast 显示错误信息
  ↓
用户看到 Toast 提示：[HTTP {status}] [Code {code}] {message}
```

## 已知问题

1. **HTTP 状态码不符合 RESTful 规范**
   - 大部分错误统一返回 500，应该根据业务错误码映射到对应的 HTTP 状态码（400、401、403、404、500 等）
   - 创建资源时应返回 201 Created，但目前返回 200 OK

2. **业务错误码使用不一致**
   - 大量使用 `NotSpecified (99999)`，应该使用更具体的错误码
   - 部分错误码定义后未使用
   - 前端的业务错误码生成脚本需要手动运行，存在同步风险

3. **错误消息处理不一致**
   - 部分错误码使用前端硬编码消息，忽略后端返回的消息
   - 部分错误码添加固定前缀，限制了后端消息的灵活性

4. **错误消息国际化**
   - 前端错误消息目前为硬编码中文，未实现国际化
