# Crater 命名空间和部署组件分析

本文档详细列出了 Crater 项目使用的所有 Kubernetes 命名空间以及依赖的 Pod 组件。

## 更新记录

> 2025-12-06：feat: show user in admin node load page ([commit 3ebdd41](https://github.com/raids-lab/crater/commit/3ebdd4117fd879d04ed12c9ab4ed7915af148893))
> - 初始文档创建，系统性梳理 Crater 项目的 Kubernetes 资源架构。
> - **组件清单**：详细列出随 Helm Chart 自动安装的核心组件（Deployment, Service, Ingress 等）及其所属命名空间。
> - **外部依赖**：规范了 Prometheus Stack、GPU Operator、Volcano 等外部组件的安装方式与命名空间要求。
> - **逻辑定义**：明确了后端 `nodeclient` 中基于命名空间和 OwnerReference 的 Pod 类型判断逻辑（System/User-Job/External）。

## 前置知识：Kubernetes 资源类型和命名空间

### Kubernetes 资源类型说明

在 Kubernetes 中，所有资源（Resource）都是对象（Object），它们都属于某个命名空间（Namespace）。以下是本文档中提到的常见资源类型：

| 资源类型 | 说明 | 作用 |
|---------|------|------|
| **Namespace** | 命名空间 | 用于逻辑隔离和组织资源，类似于文件夹 |
| **Deployment** | 部署 | 管理 Pod 的副本和更新，确保指定数量的 Pod 运行 |
| **Service** | 服务 | 为 Pod 提供稳定的网络访问入口，通过标签选择器关联 Pod |
| **Pod** | 容器组 | 运行容器的最小单元，一个 Pod 可以包含一个或多个容器 |
| **PVC** (PersistentVolumeClaim) | 持久卷声明 | 申请存储资源，Pod 通过挂载 PVC 来使用持久化存储 |
| **ConfigMap** | 配置映射 | 存储非敏感的配置数据，Pod 可以挂载为文件或环境变量 |
| **Secret** | 密钥 | 存储敏感信息（如密码、证书），Pod 可以挂载为文件或环境变量 |
| **ServiceAccount** | 服务账户 | 为 Pod 提供身份标识，用于 RBAC 权限控制 |
| **Role/RoleBinding** | 角色/角色绑定 | 定义命名空间级别的权限和绑定关系 |
| **ClusterRole/ClusterRoleBinding** | 集群角色/绑定 | 定义集群级别的权限和绑定关系 |
| **CronJob** | 定时任务 | 按时间表定期执行任务 |
| **Ingress** | 入口 | 提供 HTTP/HTTPS 路由规则，将外部流量转发到集群内的 Service |

### 命名空间的作用

- **资源隔离**：不同命名空间的资源相互隔离，同名资源可以在不同命名空间中存在
- **权限控制**：可以通过 RBAC 对不同命名空间设置不同的访问权限
- **资源组织**：将相关的资源组织在一起，便于管理

**注意**：有些资源是集群级别的（Cluster-scoped），不属于任何命名空间，如：
- `ClusterRole`、`ClusterRoleBinding`
- `Node`、`PersistentVolume`
- `StorageClass`

### 组件定义位置和安装方式

Crater 的组件分为两类：

1. **随 Helm Chart 自动安装的组件**：定义在 `charts/crater/templates/` 目录下，通过 `helm install` 或 `helm upgrade` 命令自动创建
2. **需要手动安装的依赖组件**：定义在 `backend/deployments/` 目录下，需要按照 README 中的说明手动安装

## 一、组件定义位置和安装方式总结

### 1.1 随 Crater Helm Chart 自动安装的组件

**定义位置**：`charts/crater/templates/` 目录

**安装方式**：执行 `helm install crater ./charts/crater` 或 `helm upgrade crater ./charts/crater` 时自动创建

**包含的组件**：
- 所有在 `charts/crater/templates/` 目录下定义的资源
- 包括：Deployment、Service、PVC、ConfigMap、Secret、ServiceAccount、Role、RoleBinding、CronJob、Ingress 等

### 1.2 需要手动安装的依赖组件

**定义位置**：`backend/deployments/` 目录

**安装方式**：需要按照各目录下的 README.md 中的说明手动执行 Helm 安装命令

**包含的组件**：
- Prometheus Stack（`backend/deployments/prometheus-stack/`）
- GPU Operator（`backend/deployments/gpu-operator/`）
- Volcano 调度器（`backend/deployments/volcano/`）
- CloudNativePG 数据库（`backend/deployments/cloudnative-pg/`）
- OpenEBS 存储（`backend/deployments/openebs/`，可选）
- Metrics Server（`backend/deployments/metrics-server/`，可选）

## 二、Crater 核心命名空间

### 1.1 配置的命名空间（通过 Helm Chart 配置）

根据 `charts/crater/values.yaml` 和 `backend/etc/example-config.yaml`：

| 命名空间 | 用途 | 配置来源 | 是否可配置 |
|---------|------|---------|-----------|
| `crater-workspace` | 用户作业运行命名空间 | `namespaces.job` | ✅ 可配置 |
| `crater-images` | 镜像构建命名空间 | `namespaces.image` | ✅ 可配置 |
| `{{ .Release.Namespace }}` | Crater 核心组件命名空间（默认 `crater`） | Helm Release Namespace | ✅ 可配置 |

**注意**：`{{ .Release.Namespace }}` 是 Helm 模板变量，实际部署时会被替换为 Helm Release 的命名空间，默认通常是 `crater`。

### 1.2 硬编码的命名空间

在代码中发现以下硬编码的命名空间引用：

| 命名空间 | 用途 | 位置 | 说明 |
|---------|------|------|------|
| `crater` | CronJob 管理、Leader Election | `charts/crater/templates/crater-backend/serviceaccount.yaml` | 硬编码，用于 RBAC 配置 |
| `crater-system` | PostgreSQL 数据库服务 | `charts/crater/values.yaml` (backendConfig.postgres.host) | 在配置中引用，但可能不在 Helm Chart 中创建 |

## 三、Crater 核心组件部署

**说明**：以下所有组件都定义在 `charts/crater/templates/` 目录下，随 Helm Chart 自动安装。

### 3.1 部署在 `{{ .Release.Namespace }}`（默认 `crater`）的组件

| 组件名称 | 资源类型 | 定义文件 | 说明 |
|---------|---------|---------|------|
| `crater-backend` | Deployment | `crater-backend/deployment.yaml` | 后端服务，运行 crater-backend 容器 |
| `crater-frontend` | Deployment | `crater-frontend/deployment.yaml` | 前端服务，运行 crater-frontend 容器 |
| `crater-backend-svc` | Service | `crater-backend/service.yaml` | 后端服务，提供 ClusterIP 访问 |
| `crater-frontend-svc` | Service | `crater-frontend/service.yaml` | 前端服务，提供 ClusterIP 访问 |
| `crater-backend` | ServiceAccount | `crater-backend/serviceaccount.yaml` | 后端服务账户，用于 RBAC 权限控制 |
| `crater-admin-secret` | Secret | `crater-backend/deployment.yaml` | 存储管理员初始用户名和密码 |
| `crater-backend-config` | ConfigMap | `crater-backend/configmap.yaml` | 后端配置文件 |
| `crater-frontend-config` | ConfigMap | `crater-frontend/configmap.yaml` | 前端配置文件 |
| `crater-cronjob-admin` | Role/RoleBinding | `crater-backend/serviceaccount.yaml` | CronJob 管理权限（在 `crater` 命名空间） |
| `crater-leader-election` | Role/RoleBinding | `crater-backend/serviceaccount.yaml` | Leader Election 权限（在 `crater` 命名空间） |
| `crater-viewer` | ClusterRole/ClusterRoleBinding | `crater-backend/serviceaccount.yaml` | 集群级别查看权限 |
| `cronjob-db-backup` | CronJob | `cronjob/cronjob-db-backup.yaml` | 数据库备份定时任务 |
| `cronjob-configmap` | ConfigMap | `cronjob/cronjob-configmap.yaml` | CronJob 配置 |
| `db-backup-pvc` | PVC | `cronjob/db-backup-pvc.yaml` | 数据库备份存储卷 |
| `ingress-frontend` | Ingress | `ingress/ingress-frontend.yaml` | 前端 HTTP/HTTPS 入口 |
| `ingress-backend` | Ingress | `ingress/ingress-backend.yaml` | 后端 HTTP/HTTPS 入口（如果启用） |
| `ingress-grafana-proxy` | Ingress | `ingress/ingress-grafana-proxy.yaml` | Grafana 代理入口（如果启用） |
| `ingress-storage` | Ingress | `ingress/ingress-storage.yaml` | 存储服务入口（如果启用） |
| `grafana-proxy` | Deployment/Service | `grafana-proxy/` | Grafana 代理服务（如果启用） |
| `buildkit` | StatefulSet | `buildkit/statefulset-amd.yaml`, `buildkit/statefulset-arm.yaml` | 镜像构建服务（如果启用） |

### 3.2 部署在 `crater-workspace` 命名空间的组件

| 组件名称 | 资源类型 | 定义文件 | 说明 |
|---------|---------|---------|------|
| 用户作业 Pod | Pod | 由后端动态创建 | VolcanoJob 创建的作业 Pod，通过 Kubernetes API 动态创建 |
| `webdav-deployment` | Deployment | `storage-server/deployment.yaml` | 存储服务器（WebDAV），提供文件存储服务 |
| `webdav-deployment` | Service | `storage-server/service.yaml` | 存储服务器服务，提供 ClusterIP 访问 |
| `ss-config` | ConfigMap | `storage-server/configmap.yaml` | 存储服务器配置文件 |
| `crater-rw-storage` | PVC | `storage-server/rwxpvc.yaml` | 共享存储卷（ReadWriteMany），用于存储用户数据 |
| `unified-jupyter-start-configmap` | ConfigMap | `job/unified-start-configmap.yaml` | 作业统一启动脚本配置 |
| `custom-start-configmap` | ConfigMap | `job/custom-start-configmap.yaml` | 自定义作业启动脚本配置 |
| `crater-role` | Role/RoleBinding | `crater-backend/serviceaccount.yaml` | 作业命名空间权限，允许后端在作业命名空间中创建资源 |

### 3.3 部署在 `crater-images` 命名空间的组件

| 组件名称 | 资源类型 | 定义文件 | 说明 |
|---------|---------|---------|------|
| 镜像构建 Pod | Pod | 由后端动态创建 | 镜像构建任务创建的 Pod（如 nerdctl、buildx、envd 等），通过 Kubernetes API 动态创建 |

**注意**：`crater-images` 命名空间主要用于隔离镜像构建任务，构建任务完成后 Pod 会被清理。

## 四、外部依赖组件和命名空间

**重要说明**：以下组件**不会**随 Crater Helm Chart 自动安装，需要按照 `backend/deployments/` 目录下各子目录的 README.md 说明**手动安装**。

### 4.1 监控相关（Prometheus Stack）

**定义位置**：`backend/deployments/prometheus-stack/`

**安装方式**：手动执行 Helm 安装命令（参考 `backend/deployments/prometheus-stack/README.md`）

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --version 66.2.1 \
  -f backend/deployments/prometheus-stack/kube-prometheus-stack/values.yaml \
  -n monitoring \
  --create-namespace
```

| 命名空间 | 组件 | 资源类型 | 说明 |
|---------|------|---------|------|
| `monitoring` | Prometheus | Deployment/Service | 指标收集服务 |
| `monitoring` | Grafana | Deployment/Service | 监控可视化 |
| `monitoring` | kube-state-metrics | Deployment/Service | Kubernetes 状态指标 |
| `monitoring` | AlertManager | Deployment/Service | 告警管理 |

**配置引用**：
- `backendConfig.prometheusAPI`: 后端配置中引用 Prometheus API 地址
- `grafanaProxy.address`: 在 `values.yaml` 中配置为 `http://prometheus-grafana.monitoring`

### 4.2 GPU 相关（NVIDIA GPU Operator）

**定义位置**：`backend/deployments/gpu-operator/`

**安装方式**：手动执行 Helm 安装命令（参考 `backend/deployments/gpu-operator/README.md`）

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm upgrade --install nvdp nvidia/gpu-operator \
  -f backend/deployments/gpu-operator/values.yaml \
  -n nvidia-gpu-operator \
  --create-namespace
```

| 命名空间 | 组件 | 资源类型 | 说明 |
|---------|------|---------|------|
| `nvidia-gpu-operator` | nvidia-device-plugin-daemonset | DaemonSet | GPU 设备插件，在每个节点上运行 |
| `nvidia-gpu-operator` | dcgm-exporter | DaemonSet/Service | GPU 指标导出器，用于 Prometheus 收集 GPU 指标 |
| `nvidia-gpu-operator` | gpu-operator | Deployment | GPU 操作器控制器 |
| `nvidia-gpu-operator` | node-feature-discovery | DaemonSet | 节点特性发现，自动检测节点 GPU 能力 |

**代码引用**：
- `backend/pkg/crclient/nodeclient.go` 中的 `isMonitoringPod` 函数识别 `nvidia-gpu-operator` 命名空间

### 4.3 调度器相关（Volcano）

**定义位置**：`backend/deployments/volcano/`

**安装方式**：手动执行 Helm 安装命令（参考 `backend/deployments/volcano/README.md`）

```bash
helm repo add volcano-sh https://volcano-sh.github.io/helm-charts
helm upgrade --install volcano volcano-sh/volcano \
  --namespace volcano-system \
  --create-namespace \
  --version 1.10.0 \
  -f backend/deployments/volcano/values.yaml
```

| 命名空间 | 组件 | 资源类型 | 说明 |
|---------|------|---------|------|
| `volcano-system` | volcano-scheduler | Deployment | 作业调度器，负责调度 VolcanoJob |
| `volcano-system` | volcano-controllers | Deployment | 控制器，管理 VolcanoJob 生命周期 |

**重要**：Volcano 是 Crater 的核心依赖，用于调度和管理用户作业。

### 4.4 数据库相关（CloudNativePG）

**定义位置**：`backend/deployments/cloudnative-pg/`

**安装方式**：手动执行 Helm 安装命令（参考 `backend/deployments/cloudnative-pg/README.md`）

```bash
# 1. 安装 CloudNativePG Operator
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg cnpg/cloudnative-pg \
  --namespace crater \
  --create-namespace \
  --set config.clusterWide=false \
  -f backend/deployments/cloudnative-pg/values.yaml

# 2. 安装 PostgreSQL 集群
helm pull cnpg/cluster --untar --untardir ./cluster
# 修改 cluster/templates/tests/ping.yaml（参考 README）
helm upgrade --install database \
  --namespace crater \
  ./cluster \
  -f backend/deployments/cloudnative-pg/cluster.values.yaml
```

| 命名空间 | 组件 | 资源类型 | 说明 |
|---------|------|---------|------|
| `crater` 或 `crater-system` | crater-postgresql | Cluster (Custom Resource) | PostgreSQL 数据库集群，由 CloudNativePG Operator 管理 |

**配置引用**：
- `backendConfig.postgres.host`: 在 `values.yaml` 中配置为 `crater-postgresql.crater-system.svc.cluster.local`（需要根据实际部署调整）

**注意**：PostgreSQL 可以部署在 `crater` 命名空间或独立的 `crater-system` 命名空间，取决于部署配置。

### 4.5 存储相关（OpenEBS，可选）

**定义位置**：`backend/deployments/openebs/`

**安装方式**：手动执行 Helm 安装命令（参考 `backend/deployments/openebs/README.md`，如果存在）

| 命名空间 | 组件 | 资源类型 | 说明 |
|---------|------|---------|------|
| `openebs` | OpenEBS 存储组件 | Deployment/DaemonSet | 可选，用于动态存储供应，提供 StorageClass |

**注意**：OpenEBS 是可选的，如果使用其他存储方案（如 NFS、CephFS），可以不安装。

### 4.6 Metrics Server（可选）

**定义位置**：`backend/deployments/metrics-server/`

**安装方式**：手动执行 Helm 安装命令（参考 `backend/deployments/metrics-server/README.md`，如果存在）

**说明**：Metrics Server 用于收集节点和 Pod 的资源使用情况，某些 Kubernetes 功能（如 HPA）需要它。

### 4.7 Kubernetes 系统命名空间

这些命名空间由 Kubernetes 集群自动创建，不属于 Crater 管理：

| 命名空间 | 说明 |
|---------|------|
| `kube-system` | Kubernetes 核心系统组件（如 kube-proxy、kube-dns 等） |
| `kube-public` | 公共资源，所有用户可读 |
| `kube-node-lease` | 节点租约，用于节点心跳检测 |

**代码识别**：在 `backend/pkg/crclient/nodeclient.go` 的 `determinePodType` 函数中被识别为系统 Pod

## 五、Pod 类型判断逻辑

当前 `determinePodType` 函数的判断逻辑：

1. **Kubernetes 系统命名空间** (`kube-system`, `kube-public`, `kube-node-lease`) → `"system"`
2. **Crater 镜像命名空间** (`crater-images`) → `"system"` ✅ 已修复
3. **Crater 作业命名空间** (`crater-workspace`)：
   - 有 VolcanoJob ownerReference → `"user-job"`
   - 没有 VolcanoJob ownerReference → `"system"`
4. **其他命名空间** → `"external"`

## 六、安装顺序建议

### 6.1 必需组件的安装顺序

1. **Kubernetes 集群**（已存在）
2. **Volcano 调度器**（必需）
   ```bash
   helm upgrade --install volcano volcano-sh/volcano ... -f backend/deployments/volcano/values.yaml
   ```
3. **PostgreSQL 数据库**（必需）
   ```bash
   # 安装 CloudNativePG Operator
   # 安装 PostgreSQL 集群
   ```
4. **Crater Helm Chart**（核心）
   ```bash
   helm install crater ./charts/crater -n crater --create-namespace
   ```

### 6.2 可选组件的安装顺序

可以在安装 Crater 之前或之后安装：

- **Prometheus Stack**（推荐，用于监控）
- **GPU Operator**（如果使用 GPU，推荐安装）
- **OpenEBS**（如果使用动态存储，可选）
- **Metrics Server**（如果集群没有，可选）

## 七、潜在问题和建议

### 5.1 硬编码命名空间问题

1. **`crater` 命名空间硬编码**：
   - 位置：`charts/crater/templates/crater-backend/serviceaccount.yaml` (第 73, 83, 97, 107 行)
   - 问题：CronJob 管理和 Leader Election 的 Role/RoleBinding 硬编码为 `crater` 命名空间
   - 建议：应使用 `{{ .Release.Namespace }}` 替代硬编码的 `crater`

2. **`crater-system` 命名空间引用**：
   - 位置：`charts/crater/values.yaml` (第 192 行)
   - 问题：PostgreSQL 服务地址引用 `crater-postgresql.crater-system.svc.cluster.local`，但 Helm Chart 可能不创建此命名空间
   - 建议：确认 PostgreSQL 的实际部署命名空间，或通过配置项指定

### 5.2 监控命名空间识别

在 `backend/pkg/crclient/nodeclient.go` 的 `isMonitoringPod` 函数中，识别了以下监控相关命名空间：
- `gpu-operator`
- `monitoring`
- `prometheus`
- `kube-system`
- `nvidia-gpu-operator`

但这些命名空间的 Pod 在 `determinePodType` 中仍会被标记为 `"external"`，可能需要考虑是否应标记为 `"system"`。

### 5.3 建议的命名空间分类

建议将以下命名空间的 Pod 也识别为系统 Pod：

1. **监控相关命名空间**：
   - `monitoring`（Prometheus Stack）
   - `nvidia-gpu-operator`（GPU 监控）

2. **调度器相关命名空间**：
   - `volcano-system`（Volcano 调度器）

3. **数据库相关命名空间**：
   - `crater-system`（如果 PostgreSQL 部署在此命名空间）

4. **存储相关命名空间**（可选）：
   - `openebs`（如果使用 OpenEBS）

## 八、总结

### 8.1 Crater 直接管理的命名空间（随 Helm Chart 自动创建）

- `{{ .Release.Namespace }}`（默认 `crater`）：核心组件
- `crater-workspace`：用户作业
- `crater-images`：镜像构建

### 8.2 Crater 依赖的外部命名空间（需要手动安装）

- `monitoring`：Prometheus Stack
- `nvidia-gpu-operator`：GPU Operator
- `volcano-system`：Volcano 调度器
- `crater-system` 或 `crater`：PostgreSQL 数据库（取决于部署配置）
- `kube-system`, `kube-public`, `kube-node-lease`：Kubernetes 系统命名空间

### 8.3 组件定义位置总结

| 组件类型 | 定义位置 | 安装方式 |
|---------|---------|---------|
| Crater 核心组件 | `charts/crater/templates/` | 随 Helm Chart 自动安装 |
| Prometheus Stack | `backend/deployments/prometheus-stack/` | 手动安装 |
| GPU Operator | `backend/deployments/gpu-operator/` | 手动安装 |
| Volcano | `backend/deployments/volcano/` | 手动安装 |
| PostgreSQL | `backend/deployments/cloudnative-pg/` | 手动安装 |
| OpenEBS | `backend/deployments/openebs/` | 手动安装（可选） |
| Metrics Server | `backend/deployments/metrics-server/` | 手动安装（可选） |

### 8.4 需要进一步确认的问题

1. PostgreSQL 实际部署在哪个命名空间？
2. 是否还有其他 crater 相关的命名空间需要识别为系统 Pod？
3. 监控和调度器相关的 Pod 是否应该标记为系统 Pod？

