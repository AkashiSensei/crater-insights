# Pod 配置与准备

## 更新记录

> **2026-01-15** | [refactor: streamline cron job management and introduce GPU analysis job (#323)](https://github.com/raids-lab/crater/commit/02475e3376d9e8a4553a38740d521d99b6e1b507) | `02475e3`  
> 新增节点亲和性章节，详细说明 Crater 平台中 Pod 节点亲和性的生成机制和调度逻辑。

## 节点亲和性

节点亲和性（Node Affinity）是 Pod 的调度规则，用于控制 Pod 调度到哪些节点上。Crater 平台会根据用户指定的节点选择器和镜像支持的架构自动生成节点亲和性规则，确保 Pod 调度到合适的节点。

### 节点亲和性的生成流程

节点亲和性的生成分为两个步骤：

1. **生成基础亲和性**：根据用户指定的节点选择器和资源请求生成基础节点亲和性
2. **添加架构亲和性**：根据镜像支持的架构添加架构相关的节点选择器

最终生成的节点亲和性会合并这两部分内容，设置到 Pod 的 `spec.affinity.nodeAffinity` 字段中。

### 基础亲和性的生成

`GenerateNodeAffinity` 函数（`backend/internal/handler/vcjob/util.go`）根据用户指定的选择器和资源请求生成基础节点亲和性：

- **用户指定了节点选择器**：创建硬性要求（`RequiredDuringSchedulingIgnoredDuringExecution`），Pod 必须调度到满足所有选择器条件的节点：
  - 用户可以通过前端界面（`frontend/src/components/form/other-options-form-field.tsx`）启用"指定工作节点"选项并输入节点名称，系统会创建 `kubernetes.io/hostname` 选择器，将 Pod 直接调度到指定的目标节点
- **用户未指定选择器**：根据资源请求创建软性偏好（`PreferredDuringSchedulingIgnoredDuringExecution`）：
  - 无 GPU 请求：偏好没有 GPU 和 InfiniBand 的节点（权重 40）
  - 1 个 GPU 请求：偏好没有 InfiniBand 的节点（权重 50）
  - 多个 GPU 请求：不设置偏好

软性偏好不会强制要求，但调度器会尽量满足这些条件。

### 架构亲和性的生成

`GenerateArchitectureNodeAffinity` 函数（`backend/internal/handler/vcjob/util.go`）根据镜像支持的架构添加架构相关的节点选择器：

1. **架构类型识别**：`DetermineArchitectureType` 函数（`backend/internal/handler/vcjob/util.go`）分析镜像架构列表（`archs`），识别为以下类型之一：
   - `ArchTypeAMD`：仅支持 AMD64/x86_64 架构
   - `ArchTypeARM`：仅支持 ARM64 架构
   - `ArchTypeMulti`：同时支持 AMD64 和 ARM64 架构

2. **架构选择器的添加**：
   - **单架构镜像**：添加架构限制到节点亲和性中
     - AMD64 镜像：添加 `kubernetes.io/arch=amd64` 要求
     - ARM64 镜像：添加 `kubernetes.io/arch=arm64` 要求
   - **多架构镜像**：不添加架构限制，允许调度到任何架构的节点，由容器运行时自动选择匹配的镜像版本

### 节点亲和性的合并逻辑

架构选择器会追加到用户选择器的 `MatchExpressions` 中，形成合并后的节点亲和性：

- 同一个 `NodeSelectorTerm` 内的多个 `MatchExpressions` 是 **AND 关系**（必须全部满足）
- 不同 `NodeSelectorTerm` 之间是 **OR 关系**（满足任意一个即可）

例如，用户指定了节点名称为 `node-01`（通过前端界面），镜像架构是 `amd64`，最终生成的节点亲和性要求节点同时满足：
- `kubernetes.io/hostname=node-01` **AND** `kubernetes.io/arch=amd64`

如果用户指定了其他节点标签选择器（如 `gpu-type=nvidia`），也会与架构要求合并为 AND 关系。

如果用户指定的选择器与架构要求冲突（例如用户要求 `arch=arm64` 但镜像只支持 `amd64`），Pod 将无法调度。

### 不同作业类型的处理

所有作业类型都遵循相同的节点亲和性生成流程：先调用 `GenerateNodeAffinity` 函数（`backend/internal/handler/vcjob/util.go`）生成基础亲和性，再调用 `GenerateArchitectureNodeAffinity` 函数（`backend/internal/handler/vcjob/util.go`）添加架构亲和性。不同类型的作业在节点亲和性设置上的差异主要体现在应用范围：

- **TensorFlow/PyTorch 作业**：每个任务（Task）可以有不同的镜像，系统会为每个任务单独生成节点亲和性，确保每个任务调度到匹配其镜像架构的节点。在 `backend/internal/handler/vcjob/tensorflow.go` 和 `backend/internal/handler/vcjob/pytorch.go` 中，首先为整个作业生成基础亲和性（基于用户选择器和资源请求），然后为每个任务的镜像调用 `GenerateArchitectureNodeAffinity` 函数生成任务级别的节点亲和性
- **Custom 训练作业**：单个 Pod，直接根据镜像架构和用户选择器生成节点亲和性。通过 `GenerateCustomPodSpec` 函数生成 Pod Spec，先调用 `GenerateNodeAffinity` 生成基础亲和性，再调用 `GenerateArchitectureNodeAffinity` 添加架构亲和性
- **Jupyter/WebIDE 交互式作业**：使用 `generateInteractivePodSpec` 函数（`backend/internal/handler/vcjob/util.go`）生成 Pod Spec，同样先调用 `GenerateNodeAffinity` 生成基础亲和性，再调用 `GenerateArchitectureNodeAffinity` 添加架构亲和性

所有作业类型最终都会将生成的节点亲和性设置到 Pod Spec 的 `affinity.nodeAffinity` 字段中，由 Volcano 调度器根据这些规则进行调度。

### 多架构镜像的调度策略

对于支持多个架构的镜像（如同时支持 `linux/amd64` 和 `linux/arm64`），系统采用**不限制架构**的策略：

- 不添加架构相关的节点选择器
- 保留用户指定的节点选择器（如果有）
- 允许 Pod 调度到任何架构的节点

这种设计让多架构镜像的调度更加灵活，容器运行时（如 containerd）会根据节点架构自动拉取匹配的镜像版本。但需要注意，如果镜像不支持某个架构（如 sw64），Pod 可能调度到不支持的节点，导致运行时失败。

这个设计其实是相对丑陋的，在有超过两种架构时很容易产生问题。