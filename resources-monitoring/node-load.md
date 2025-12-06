# 节点负载

监控在节点上运行的 Pod。

## 更新记录

> 2025-12-06：feat: show user in admin node load page ([commit 3ebdd41](https://github.com/raids-lab/crater/commit/3ebdd4117fd879d04ed12c9ab4ed7915af148893))
> - 初始文档创建，完整记录节点负载功能的前后端实现机制。
> - **后端架构**：详述基于 controller-runtime 索引器的 Pod 查询优化方案、API 接口分层（用户/管理员），以及基于 `omitempty` 的敏感字段（用户/账户信息）权限控制策略。
> - **前端交互**：说明 React Query 数据获取流程、Pod 类型动态生成逻辑，以及表格列（名称、资源、归属等）的详细渲染规则与 UI 语义化实现。
> - **核心机制**：解释 Request 资源计算方法、Volcano Job 锁定状态查询及 Pod 类型分类逻辑。

## 概述

节点负载功能用于展示指定节点上运行的所有 Pod 信息，包括系统 Pod、用户作业 Pod 和外部 Pod。该功能通过 Kubernetes API 获取 Pod 数据，并使用索引器优化查询性能。

## 架构流程

### 1. 后端数据获取

#### 1.1 索引器设置

**位置**：`backend/pkg/indexer/indexer.go`

**索引器概述**：索引器索引的是 controller-runtime 缓存中的全部 Pod 数据（来自 Kubernetes API Server），目的是优化按节点名称查询 Pod 的性能，避免全量扫描所有 Pod。索引器基于 controller-runtime 的缓存机制，在后端系统启动时建立索引规则，索引数据在缓存同步时自动构建。

**索引规则**：为 Pod 建立 `spec.nodeName` 字段的索引，建立 `节点名称 -> Pod 列表` 的映射关系。索引建立时会排除以下 Pod：
- `PodFailed` 和 `PodSucceeded` 状态的 Pod（这些 Pod 的数据仍存在于 Kubernetes 集群中，只是不会被索引，因此通过索引器查询时不会返回）
- `spec.nodeName` 为空的 Pod（未调度到节点的 Pod）

**实现**：

```go
// SetupIndexers 设置索引器
func SetupIndexers(mgr manager.Manager) error {
    return mgr.GetFieldIndexer().IndexField(context.Background(), &corev1.Pod{}, PodNodeNameIndex, func(rawObj client.Object) []string {
        pod := rawObj.(*corev1.Pod)
        if pod.Status.Phase == corev1.PodFailed || pod.Status.Phase == corev1.PodSucceeded {
            return nil  // 不建立索引
        }
        if pod.Spec.NodeName == "" {
            return nil  // 不建立索引
        }
        return []string{pod.Spec.NodeName}  // 建立索引
    })
}
```

**调用时机**：在 `backend/cmd/crater/helper/manager.go` 的 `SetupCustomCRDAddon` 方法中调用，后端系统启动时执行。

#### 1.2 API 接口

**位置**：`backend/internal/handler/node.go`

**接口定义**：
- **用户接口**：`GET /api/v1/nodes/{name}/pods`（`RegisterProtected`），调用 `GetPodsForNode` 处理函数
- **管理员接口**：`GET /api/v1/admin/nodes/{name}/pods`（`RegisterAdmin`），调用 `AdminGetPodsForNode` 处理函数

**逻辑复用**：
为了减少代码重复，后端引入了 `handleGetPods` 辅助方法，封装了参数解析、日志记录和错误处理的通用逻辑。`GetPodsForNode` 和 `AdminGetPodsForNode` 仅需传入对应的 Client 方法（`fetcher`）即可。

#### 1.3 数据获取逻辑

**位置**：`backend/pkg/crclient/nodeclient.go`

**用户接口方法**：`GetPodsForNode` 通过索引器查询指定节点上的 Pod，转换数据结构，并对 VCJob Pod 查询数据库获取锁定状态。此接口**不填充**用户和账户相关的敏感字段。

**管理员接口方法**：`AdminGetPodsForNode` 通过索引器查询指定节点上的 Pod，判断 Pod 类型，对 user-job 类型的 Pod 查询数据库并**预加载（Preload）**作业、用户和账户信息。

**流程**：

1. **使用索引器查询 Pod**：通过 `indexer.MatchingPodsByNodeName(nodeName)` 创建查询条件，利用已建立的索引快速定位指定节点上的 Pod。返回的 Pod 列表只包含已建立索引的 Pod（自动排除 `Failed` 和 `Succeeded` 状态的 Pod）。

2. **转换 Pod 数据结构**：将 Kubernetes Pod 对象转换为前端需要的格式，包括名称、命名空间、IP、创建时间、状态、资源申请量等字段。资源申请量通过 `CalculateRequsetsByContainers` 函数计算，累加 Pod 中所有容器的 `Resources.Requests`（即 Kubernetes 的 request 资源，而非 limit）。

3. **用户接口特殊处理**：对于 VCJob 类型的 Pod（`batch.volcano.sh/v1alpha1/Job`），查询数据库获取作业的锁定状态信息，设置 `Locked`、`PermanentLocked`、`LockedTimestamp` 字段。其他 Pod 的锁定状态保持为 `false`。

4. **管理员接口特殊处理**：
   - **判断 Pod 类型**：通过 `determinePodType` 函数判断 Pod 类型（user-job、system、external）
   - **查询作业和用户信息**：对于 `user-job` 类型的 Pod，从 `ownerReference` 中提取作业名称，查询数据库（使用 `Preload` 加载 User 和 Account 关联表）获取完整信息
   - **设置管理员字段**：设置 `JobName`、`UserName`、`UserRealName`、`AccountName`、`AccountRealName` 等字段

**Pod 数据结构与字段级权限控制**：

利用 Go 语言结构体的 `omitempty` 标签，实现字段级的权限控制。敏感字段（如用户信息）仅在管理员接口逻辑中被填充，在用户接口中保持零值，从而在 JSON 序列化时被自动忽略。

```go
type Pod struct {
    Name            string                  `json:"name"`
    Namespace       string                  `json:"namespace"`
    OwnerReference  []metav1.OwnerReference `json:"ownerReference"`
    IP              string                  `json:"ip"`
    CreateTime      metav1.Time             `json:"createTime"`
    Status          corev1.PodPhase         `json:"status"`
    Resources       corev1.ResourceList     `json:"resources"`
    Locked          bool                    `json:"locked"`
    PermanentLocked bool                    `json:"permanentLocked"`
    LockedTimestamp metav1.Time             `json:"lockedTimestamp"`
    
    // 管理员接口返回的字段（omitempty 表示字段为空时不序列化）
    JobName         string `json:"jobName,omitempty"`         // 作业名称
    UserName        string `json:"userName,omitempty"`        // 用户昵称（用于显示）
    UserID          uint   `json:"userID,omitempty"`          // 用户ID（用于跳转）
    UserRealName    string `json:"userRealName,omitempty"`    // 用户真实名称（用于tooltip）
    AccountName     string `json:"accountName,omitempty"`     // 账户昵称（用于显示）
    AccountID       uint   `json:"accountID,omitempty"`       // 账户ID（用于跳转）
    AccountRealName string `json:"accountRealName,omitempty"` // 账户真实名称（用于tooltip）
    PodType         string `json:"podType,omitempty"`         // Pod类型：user-job, system, external
}
```

#### 1.4 数据筛选

**筛选机制**：索引器在建立索引时进行筛选，排除 `Failed` 和 `Succeeded` 状态的 Pod 以及未调度的 Pod。通过索引器查询时，只返回已建立索引且节点名称匹配的 Pod。被排除的 Pod 数据仍然存在于 Kubernetes 集群中，只是通过索引器查询时不会返回。

**返回范围**：
- ✅ **状态筛选**：只返回活跃状态的 Pod（`Running`、`Pending` 等）
- ✅ **节点筛选**：只返回指定节点上的 Pod
- ❌ **类型筛选**：无，返回所有类型的 Pod（系统、外部、用户作业）
- ❌ **命名空间筛选**：无，返回所有命名空间的 Pod

**返回的 Pod 类型**：
- **系统 Pod**：kube-system、kube-public、kube-node-lease 等系统命名空间的 Pod
- **外部 Pod**：其他命名空间的 Pod（如 crater、metallb-system、prometheus 等）
- **用户作业 Pod**：Volcano Job 创建的 Pod（在作业命名空间中）

**特殊处理**：
- 只有 **VCJob Pod**（`batch.volcano.sh/v1alpha1/Job`）会查询数据库获取锁定状态
- 其他 Pod 的 `Locked` 字段保持为 `false`

### 2. 前端数据展示

#### 2.1 API 调用

**位置**：`frontend/src/components/node/node-detail.tsx`

**数据查询**：使用 React Query 根据用户角色选择不同的 API 接口：
- **管理员视图**：调用 `apiAdminGetNodePods` 接口（`GET /api/v1/admin/nodes/{name}/pods`），返回包含作业和用户信息的 Pod 列表
- **用户视图**：调用 `apiGetNodePods` 接口（`GET /api/v1/nodes/{name}/pods`），返回基础 Pod 信息

对返回的数据进行排序，并根据 `ownerReference` 动态生成 `type` 字段（格式：`apiVersion/kind`）用于前端筛选。

**数据接口**：`frontend/src/services/api/cluster.ts`

```typescript
export interface IClusterPodInfo {
  // ... 基础字段 ...
  // 管理员接口返回的字段
  jobName?: string
  userName?: string
  userID?: number
  userRealName?: string
  accountName?: string
  accountID?: number
  accountRealName?: string
  podType?: string
}
```

#### 2.2 表格列定义

**位置**：`frontend/src/components/node/node-detail.tsx`

**列配置**：表格包含以下列：

1. **类型**（`type`）：
   - **数据来源**：前端根据 Pod 的 `ownerReference` 动态生成，格式为 `apiVersion/kind`
   - **定义来源**：`ownerReference` 是 Kubernetes 的标准概念，定义在 Kubernetes API 中（`metav1.OwnerReference`），用于表示 Pod 的所有者资源。这是 Kubernetes 原生的字段，不是项目自定义的
   - **生成逻辑**：前端在数据查询后，遍历每个 Pod，如果 `ownerReference` 数组不为空，取第一个元素的 `apiVersion` 和 `kind` 组合成 `type` 字段（格式：`${ownerReference[0].apiVersion}/${ownerReference[0].kind}`）
   - **常见类型**：
     - `batch.volcano.sh/v1alpha1/Job`：Volcano Job（Crater 使用的作业类型，由 Volcano 控制器创建）
     - `aisystem.github.com/v1alpha1/AIJob`：EMIAS AIJob（Crater 使用的作业类型，由 EMIAS 控制器创建）
     - `apps/v1/ReplicaSet`：ReplicaSet（通常由 Deployment 创建，用于管理 Pod 副本）
     - `apps/v1/DaemonSet`：DaemonSet（系统组件，确保每个节点运行一个 Pod 副本）
     - `apps/v1/StatefulSet`：StatefulSet（有状态应用，如数据库）
     - `batch/v1/Job`：Kubernetes 原生 Job（一次性任务）
     - 无 OwnerReference 的 Pod：该列为空（如直接创建的 Pod 或某些系统 Pod）
   - **显示方式**：Badge 组件显示 `kind` 部分（如 "Job"、"ReplicaSet"），hover 时显示完整的 `apiVersion`（如 "batch.volcano.sh/v1alpha1"）

2. **命名空间**（`namespace`）：Pod 的 `metadata.namespace`。

3. **名称**（`name`）：
   - **数据来源**：Pod 的 `metadata.name` 字段（Kubernetes 标准字段）
   - **显示逻辑**：
     - **主标题**：如果 Pod 有 Job 类型的 `ownerReference`（`kind === 'Job'`），显示作业名称（`ownerReference[0].name`），否则显示 Pod 名称（`metadata.name`）
     - **副标题**：如果主标题显示的是作业名称，则在下方以小字显示 Pod 名称（`metadata.name`）
   - **关系说明**：
     - **Pod 与 Job 的关系**：Pod 由 Job 创建，Job 是 Pod 的所有者（Owner）。这是 Kubernetes 的标准资源关系，通过 `ownerReference` 字段建立
     - **命名规则**：
       - **作业名称**：由用户或系统指定，具有业务含义（如用户创建的作业名称）
       - **Pod 名称**：由 Job 控制器自动生成，通常格式为 `{job-name}-{随机后缀}`（如 `my-job-abc123`），用于唯一标识每个 Pod 实例
     - **为什么显示作业名称**：一个 Job 可以创建多个 Pod（并行任务），每个 Pod 的名称不同但都属于同一个作业。主标题显示作业名称更便于识别和管理，副标题显示 Pod 名称用于精确标识具体的 Pod 实例
   - **交互**：点击可查看 Pod 监控，锁定状态的 Pod 会显示锁定图标

4. **状态**（`status`）：
   - **数据来源**：Pod 的 `status.phase` 字段（Kubernetes 标准字段）
   - **可能值**：`Pending`（等待调度）、`Running`（运行中）、`Succeeded`（成功完成）、`Failed`（失败）、`Unknown`（未知状态）
   - **显示方式**：使用 `PodPhaseLabel` 组件，带颜色标识（如 Running 为绿色，Failed 为红色）

5. **申请资源**（`resources`）：显示 Request 资源。

6. **创建于**（`createTime`）：使用 `TimeDistance` 组件。

7. **操作**（`actions`）：监控和日志查看功能。

8. **归属**（管理员视图）：
   - **内容**：包含“所属用户”和“所属账户”信息。
   - **展示逻辑**：
     - 与“名称”列保持一致的 UI 结构（`SimpleTooltip` + `span` + `onClick`）。
     - **国际化（i18n）**：Tooltip 中的文本（如 "View info of {user}"）包含变量和样式嵌套，使用 `Trans` 组件实现，支持中、英、日、韩四种语言的准确翻译和语序调整。
     - 仅在有对应数据时渲染，否则显示 "-" 或仅显示文本。

#### 2.3 筛选配置

**位置**：`frontend/src/components/node/node-detail.tsx`

**筛选器**：提供名称搜索、命名空间筛选、状态筛选和类型筛选。状态筛选默认显示 `Running` 和 `Pending` 状态的 Pod。

#### 2.4 关键实现细节

**UI 一致性与语义化**
在表格列的实现中，不同状态（可点击 vs 不可点击）和不同内容（带图标 vs 纯文本）往往会导致细微的视觉错位。解决方案是：
1.  **统一容器**：无论内容如何，外层都使用结构相同的 `flex` 容器。
2.  **避免非法嵌套**：`SimpleTooltip` 组件通常会渲染一个交互式的 Trigger（如 `button`）。在其中直接使用 `Link` 组件（渲染为 `a` 标签）是 HTML 语义非法的。通过使用 `span` 元素并接管 `onClick` 事件来通过编程方式导航（`useNavigate`），既保持了语义正确，又规避了浏览器对非法嵌套的自动修正导致的布局问题。

## 相关文件

### 后端
- `backend/pkg/indexer/indexer.go` - 索引器定义
- `backend/pkg/crclient/nodeclient.go` - Pod 数据获取逻辑
- `backend/internal/handler/node.go` - API 处理函数

### 前端
- `frontend/src/components/node/node-detail.tsx` - 节点详情页面组件
- `frontend/src/services/api/cluster.ts` - API 接口定义
- `frontend/src/components/query-table/index.tsx` - 数据表格组件
