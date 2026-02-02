# RDMA

RDMA 是 Crater 系统的扩展功能之一。

本文档记录系统如何配置 k8s 容器使用 RDMA 功能，以及其相关的基本组件工作原理。

## 更新记录

> **2026-01-30** | [[feat] support rdma for custom jobs (#335)](https://github.com/raids-lab/crater/commit/6fe5499f251b030862a613d9bbbed57e9002e158) | `6fe5499`
> 完善平台对 RDMA 的处理流程说明，明确支持包括 Jupyter 和单机作业在内的自定义作业类型。细化前端动态展示逻辑及作业创建时的双重注入机制，确保不同类型的作业均能通过绑定的 GPU 型号自动获取 RDMA 资源配置。

## RDMA 简介

RDMA (Remote Direct Memory Access) 是一种高性能的网络传输技术，允许计算机直接访问另一台计算机的内存，而无需经过双方操作系统的内核。在 Crater 这样的 AI 算力平台中，RDMA 是实现高效分布式训练（如 PyTorch DDP）和高性能计算的基础。

其核心特性包括：

*   **零拷贝 (Zero-Copy)**：数据直接从内存传输到网络适配器，无需在操作系统内核缓冲区和应用缓冲区之间进行多次拷贝。
*   **内核旁路 (Kernel Bypass)**：应用程序可以直接与硬件交互，避开了繁重的内核网络协议栈处理，显著降低了通信延迟。
*   **CPU 卸载 (CPU Offload)**：数据传输完全由硬件处理，不占用 CPU 资源，使 CPU 能够专注于 AI 计算任务。

### CPU RDMA vs GPU RDMA
在不同的应用场景下，RDMA 的关注点有所不同：
*   **CPU RDMA (传统模式)**：主要解决**宿主机内存 (RAM)** 之间的数据交换。
*   **GPU RDMA (GPUDirect RDMA)**：在 AI 场景下，我们通常指网卡直接访问 **GPU 显存**。数据流经由 PCIe 总线在网卡与 GPU 之间直接传输，**完全绕过 CPU 内存**，从而消除了内存拷贝的延迟并显著降低了 CPU 负载。

### 发展背景与组织

RDMA 技术最早源于 1999 年成立的 **InfiniBand Trade Association (IBTA)** 发布的 InfiniBand 架构规范。IBTA 是目前 RDMA 领域最核心的标准制定组织，由 NVIDIA (前 Mellanox)、Intel、IBM、HPE 等行业巨头共同领导。

此外，**OpenFabrics Alliance (OFA)** 也是关键组织之一，它致力于开发和维护 RDMA 的开源软件栈（即常见的 OFED）。

目前主流的 RDMA 实现方案包括：
*   **InfiniBand**：最原生的 RDMA 方案，需要专门的交换机和网卡。
*   **RoCE (RDMA over Converged Ethernet)**：在以太网上运行 RDMA 的方案，目前在数据中心 AI 算力集群中应用最为广泛。
*   **iWARP**：基于 TCP/IP 协议的 RDMA 方案。

**官方网站：**
*   IBTA 官网: [www.infinibandta.org](https://www.infinibandta.org)
*   OpenFabrics Alliance: [www.openfabrics.org](https://www.openfabrics.org)

## 软硬件需求与依赖

### 硬件需求
1.  **支持 RDMA 的网卡 (HCA)**：
    *   通常需要高性能网络适配器，如 **NVIDIA (Mellanox) ConnectX 系列**（ConnectX-5, 6, 7 等）。
    *   网卡需支持 **InfiniBand** 或 **RoCE** (RDMA over Converged Ethernet) 协议。
2.  **GPU (针对 AI 场景)**：
    *   **硬件要求**：需要 **NVIDIA 数据中心级 GPU**（如 V100, A100, H100）。
    *   **PCIe 拓扑**：为了实现最优的 GPUDirect RDMA，GPU 和网卡应当挂载在同一个 PCIe Root Complex 下，以支持 Peer-to-Peer (P2P) 传输。
3.  **交换机**：
    *   **InfiniBand**：需要专门的 IB 交换机。
    *   **RoCE**：需要支持 **无损以太网 (Lossless Ethernet)** 的交换机，并配置 PFC (Priority Flow Control) 和 ECN (Explicit Congestion Notification)。

### 软件依赖（分层配置）

#### A. 集群与宿主机层面 (Infrastructure Level)
这是平台管理员需要完成的配置，决定了集群是否“具备” RDMA 能力：
1.  **宿主机驱动 (MOFED)**：每一台物理节点必须安装与网卡型号匹配的 **Mellanox OFED 驱动**。
2.  **Kubernetes 资源发现**：
    *   部署 **NVIDIA Network Operator**。
    *   配置 **RDMA Shared Device Plugin**（或其他模式如 SR-IOV），将宿主机的 RDMA 设备作为 K8s 资源（如 `rdma/rdma_v100`）上报给 API Server。
3.  **安全策略 (Capabilities)**：
    *   由于 RDMA 需要调用 `mlock` 锁页内存，K8s 层面必须允许 Pod 开启 **`IPC_LOCK`** 能力。

#### B. 用户镜像层面 (User Image Level)
这是用户在构建 Dockerfile 时需要安装的内容，决定了应用是否能“调用” RDMA 能力：
1.  **用户态基础库**：镜像内必须安装 RDMA 的核心运行库：
    *   `libibverbs1`：访问 IB 硬件的基础库。
    *   `librdmacm1`：RDMA 连接管理库。
    *   `libibumad3`、`ibverbs-providers` 等。
2.  **调试与测试工具**：
    *   `perftest`：包含 `ib_write_bw`、`ib_send_lat` 等必备的带宽与延迟测试工具。
    *   `infiniband-diags`：包含 `ibstat`、`ibv_devinfo` 等查询设备状态的工具。
3.  **通信框架支持**：
    *   如果使用 PyTorch/TensorFlow，通常依赖 **NCCL (NVIDIA Collective Communications Library)**。镜像内的 NCCL 必须检测到系统环境支持 RDMA 才能启用。

### 平台实践参考
关于在 Crater 平台上安装相关依赖并启用 RDMA 的主要流程、配置示例及避坑指南，请参考官方文档：
*   [Crater 官方文档 - RDMA 支持](https://raids-lab.github.io/crater/zh/docs/admin/more/rdma/)

## 平台对 RDMA 的处理流程

Crater 平台通过一套自动化的资源发现与绑定机制，将 K8s 底层的物理能力平滑地对接给用户作业。

### 1. 资源发现与同步机制 (Resource Discovery)
在 K8s 集群中，GPU 和 RDMA 资源都被视为**扩展资源 (Extended Resources)**，平台的发现逻辑对二者是并列处理的：
*   **同步逻辑**：`SyncResource` 接口调用时，后端会扫描节点状态中的 `Allocatable` 字段。
*   **并列识别**：平台会同时识别出如 `nvidia.com/v100`（GPU 资源）和 `rdma/rdma_v100`（RDMA 资源）。例如，一个节点上可能同时报告有 8 个 V100 GPU 和 8 个对应的 RDMA 虚拟设备。
*   **资源分层**：在数据库中，它们都被抽象为 `Resource` 模型，通过 `ResourceType` 进行区分（`gpu`, `rdma` 等）。

### 2. 管理员绑定与拓扑映射 (Administrative Binding)
由于不同型号的 GPU（如 V100 与 A100）通常对应不同的 RDMA 网络或网卡，平台引入了“绑定”机制来描述这种拓扑对应关系：
*   **配置方式**：管理员登录 **Crater 管理后台 (Admin Dashboard)**，在“集群管理 -> 资源管理”页面进行操作：
    1.  **标记类型**：首先需要将同步上来的资源手动修改类型。例如，将 `nvidia.com/v100` 标记为 `gpu` 类型，将 `rdma/rdma_v100` 标记为 `rdma` 类型。
    2.  **建立关联**：在标记为 `gpu` 的资源行中，点击“网络关联”按钮。在弹出的界面中，系统会过滤出所有 `rdma` 类型的资源，管理员选择匹配的型号点击“连接”即可。
*   **存储形式**：绑定关系持久化存储在平台的 **关系型数据库 (Database)** 中：
    *   **关联表**：系统使用一张名为 `resource_networks` 的中间关联表（由 `ResourceNetwork` 模型定义）。
    *   **核心字段**：主要记录 `ResourceID` (GPU 资源 ID) 与 `NetworkID` (RDMA 资源 ID) 的对应关系。
*   **绑定的意义**：
    *   **决策依据**：它是 `GetGPUNetworks` 接口的核心。当前端询问“这个 GPU 型号是否支持 RDMA”时，后端会根据此数据库关联表返回对应的 RDMA 资源详情。
    *   **资源选型**：它确保了用户在选择 V100 时，系统知道应该去申请 `rdma/rdma_v100` 而不是其他的 RDMA 资源。

### 3. 前端动态展示与型号锁定
*   **动态触发**：用户在创建作业页面选择 GPU 型号后，前端会立即调用后端的关联接口。
*   **开关显示**：只有当后端返回了该 GPU 型号绑定的 RDMA 资源时，前端才会展示“开启 RDMA”的开关。
*   **作业参数生成**：开启开关后，前端提交的作业请求中将包含 `network.enabled: true`。

### 4. 作业创建时的“同步注入” (Simultaneous Injection)
当后端接收到开启 RDMA 的请求时，会在构造 Pod Spec 的瞬间完成双重注入：
*   **GPU 需求注入**：正常注入用户要求的 GPU 数量（如 `nvidia.com/v100: 1`）。
*   **RDMA 需求同步注入**：后端会根据前述的**绑定关系**，自动查找到对应的 RDMA 资源名，并以 1:1 的比例同步注入资源请求（如 `rdma/rdma_v100: 1`）。
*   **权限与调度**：
    *   注入 **`IPC_LOCK`** 权限（确保“能锁内存”）。
    *   K8s 调度器根据 `resources` 中的双重需求，确保 Pod 被分发到同时具备该型号 GPU 和 RDMA 网卡的节点上。

## 权限控制与资源限制的作用机理

在容器化环境中使用 RDMA，涉及到 Linux 内核对内存管理的深度控制。理解其机理需要从“内存固定”这一底层需求出发，逐层向上构建权限与限额的逻辑。

### 1. 根源：为什么 RDMA 需要“固定内存” (Memory Pinning)
RDMA 的核心是让硬件（网卡 HCA）绕过 CPU 直接访问远程内存。
*   **物理地址稳定性**：在普通操作中，操作系统为了优化内存，会随时进行“换页” (Paging) 或移动数据，导致虚拟地址对应的物理地址发生变化。
*   **网卡的要求**：硬件网卡仅能识别物理地址。如果网卡正在传输数据时，内核将该页内存换出或移动，会导致传输崩溃甚至系统错误。
*   **实现手段 (`mlock`)**：为了保证地址绝对稳定，RDMA 在传输前必须执行“内存注册” (Memory Registration, MR)。这一动作在内核层面通过 **`mlock()`** 或 **`mlockall()`** 系统调用实现，它们会将指定的虚拟内存页“锁死”在物理内存中，禁止内核对其进行移动或交换到磁盘。
*   **GPU RDMA 的情况**：在 **GPUDirect RDMA** 中，虽然数据流主路径在 GPU 显存内，但其**控制路径 (Control Path)** 以及相关的内核缓冲区申请依然涉及宿主机内存管理。同时，NCCL 等通信库通常会对系统环境的内存锁定限制进行统一检查。

### 2. 第一层：权限控制 (Capability - “准许证”)
有了调用 `mlock` 的需求，进程必须首先获得内核的“准许证”：
*   **`CAP_IPC_LOCK`**：这是 Linux 提供的一项内核能力。它决定了一个进程是否有权发起 `mlock()` 动作。
*   **作用机理**：当进程发起 `mlock` 系统调用时，内核会检查该进程的 `Capability` 集合。如果没有 `CAP_IPC_LOCK`，调用将立即返回 `EPERM` (Permission denied)，无论当前内存上限是多少。
*   **平台注入**：Crater 平台在用户启用 RDMA 时，会自动在 Pod 的 `securityContext.capabilities` 中注入此能力。（实际上对于所有作业，都会注入 `IPC_LOCK`）

### 3. 第二层：容量限制 (Resource Limit - “配额管理”)
即便拥有了“准许证”，内核还会根据“配额”来限制进程锁定的总内存量：
*   **`RLIMIT_MEMLOCK`**：这是内核维护的一个资源限制项，对应 `ulimit -l` 的值。它定义了一个进程及其子进程允许锁定的物理内存总量。
*   **限额的设定与修改机理**：
    *   **初始设定 (Creation Time)**：这是限额的源头。在容器启动过程中，**容器运行时 (containerd/runc)** 会在 `fork` 之后、`exec` 之前调用 `setrlimit()`，为容器进程设定 **Hard Limit（硬限制）**。这个硬限制是该容器进程及其所有后代进程无法逾越的最高天花板。
    *   **会话级修改 (Session Time)**：这是 **PAM (Pluggable Authentication Modules)** 介入的时机。当通过 `ssh` 登录或执行 `su -` 时，系统会加载 `pam_limits.so` 模块。该模块会读取 **`/etc/security/limits.conf`** 配置文件，并尝试通过 `setrlimit()` 将当前会话的限额（Soft Limit）调整为配置的值。
    *   **权限约束**：修改限额的操作受内核严格管控。进程可以自由调低硬限制，或者在硬限制范围内调高软限制；但如果想要**调高硬限制**（捅破天花板），进程必须拥有 **`CAP_SYS_RESOURCE`** 能力。
*   **机理总结**：容器内的容量环境由运行时的初始硬限制（Hard Limit）决定。若运行时设定的天花板过低，且容器进程缺乏修改天花板的特权，那么即便在 `limits.conf` 中配置了 `unlimited`，PAM 模块在尝试应用配置时也会因权限不足而被内核拦截。

## 潜在问题：1M 以上数据传输失败 ([Issue #339](https://github.com/raids-lab/crater/issues/339))

在实际部署中，用户可能会遇到如下典型故障：**小数据量（如 1024 字节）测试正常，但 1M 及以上量级传输时报错 MR 分配失败。** 即使在镜像中修改了 `limits.conf`，容器内的 `ulimit -l` 依然显示为 64KB。

根据前文所述机理，该问题的本质是 **“Hard Limit 锁定”**：
1.  **天花板过低**：容器运行时（containerd/runc）在创建容器时，将 `memlock` 的硬限制定死在 64KB。
2.  **修改无效**：由于容器进程缺乏 `CAP_SYS_RESOURCE` 权限，无论是 PAM 模块还是手动执行 `ulimit -l unlimited`，都无法突破这个 64KB 的硬天花板。
3.  **库预检失败**：NCCL 或 InfiniBand 驱动库在检测到 64KB 的上限后，预判无法支撑大数据量所需的内存注册，从而直接报错退出。

大致可以从以下三个方向解决这个问题：
- 全局修改宿主机 ulimit 限制
- 给对应容器 `CAP_SYS_RESOURCE` 权限
- 在宿主机上设置放开限制的专门 handler，需要放开限制的 Pod 指定对应的 RuntimeClass

> **注意：Kubernetes 官方设计说明**
> Kubernetes 并不直接在 Pod API (spec.containers) 中提供 `ulimit` (包括 `memlock`) 的配置项。官方认为这类系统级限制属于容器运行时（CRI）的职责范畴。因此，所有解决方案均需围绕“运行时配置”或“内核特权”展开。


### 方案 A：节点运行时配置（推荐，最根本）
在运行 RDMA 作业的物理节点上，修改容器运行时的默认资源限制：
*   **操作**：
    *   **containerd**: 修改 `/etc/containerd/config.toml`，在 `[plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]` 下设置 `systemd_cgroup = true` 并确保 `default_ulimits` 包含 `memlock` 为 `-1`。
    *   **Docker**: 修改 `/etc/docker/daemon.json` 或 systemd 服务文件，添加 `"default-ulimits": {"memlock": {"Name": "memlock", "Hard": -1, "Soft": -1}}`。
*   **风险分析**：
    *   **资源枯竭风险**：由于该配置是全局生效的，意味着该节点上**所有**新启动的容器都将拥有无限锁页内存的权限。如果某个容器内运行了恶意程序或存在内存泄漏的 Bug 进程，它可能会锁定物理节点上几乎所有的内存，导致宿主机内核（OOM Killer）甚至无法正常工作，最终引发整个物理节点宕机。
    *   **多租户干扰**：在多租户集群中，一个作业的异常可能会影响到同节点其他租户的任务稳定性。

### 方案 B：增加容器特权（不推荐，安全性差）
在 K8s Pod Spec 的 `securityContext` 中额外添加 **`CAP_SYS_RESOURCE`** 权限。
*   **效果**：赋予容器内的 root 用户调高硬限制的能力。
*   **操作**：需在容器 Entrypoint 中手动执行 `ulimit -l unlimited`。
*   **风险分析（越界行为）**：
    *   **突破系统配额**：`CAP_SYS_RESOURCE` 是 Linux 内核中最危险的特权之一。获得该权限的用户不仅可以修改 `memlock`，还可以随意调高**文件描述符上限 (nofile)**、**进程数上限 (nproc)**、甚至绕过磁盘配额检查。
    *   **拒绝服务攻击 (DoS)**：恶意用户可以利用此权限耗尽系统的所有文件句柄或进程号，导致宿主机及节点上其他容器因无法打开新文件或创建新进程而崩溃。
    *   **破坏隔离性**：这严重违反了容器的最小权限原则，使得容器不再是一个受限的安全沙箱。

### 方案 C：使用 RuntimeClass (推荐的生产级方案)
这是目前在安全性与灵活性之间取得最佳平衡的方案。其核心思想是：**由具备特权的“管家”（运行时）提前为容器破开天花板，而不是给容器内的“住户”发钥匙。**

*   **实现机理**：
    1.  **节点侧配置**：在物理节点的 `containerd` 配置文件中增加一个专门的 `rdma` handler，在该 handler 的配置中通过 `base_runtime_spec` 指向一个预定义的 OCI 规范文件。
    2.  **集群侧定义**：在 K8s 中创建一个 `RuntimeClass` 对象（如 `name: rdma-runtime`），将其 `handler` 指向节点侧配置的名称。
    3.  **作业侧引用**：Crater 平台在检测到作业启用 RDMA 时，自动在 Pod Spec 中注入 `runtimeClassName: rdma-runtime`。
*   **官方支持确认**：
    `containerd` 官方在 CRI 配置文档中明确支持了 `base_runtime_spec` 参数，该参数遵循 **OCI (Open Container Initiative) Runtime Specification** 行业标准。
    *   **containerd 配置参考**：[containerd CRI Configuration Guide](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)
    *   **OCI Runtime Spec 标准参考**：[OCI Runtime Spec - POSIX Process Limits](https://github.com/opencontainers/runtime-spec/blob/main/config.md#posix-platform-process-limits)
*   **配置示例**：
    *   **/etc/containerd/config.toml**:
        ```toml
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.rdma]
          runtime_type = "io.containerd.runc.v2"
          base_runtime_spec = "/etc/containerd/rdma-base.json"
        ```
    *   **/etc/containerd/rdma-base.json** (关键部分):
        ```json
        {
          "process": {
            "rlimits": [
              {
                "type": "RLIMIT_MEMLOCK",
                "hard": -1,
                "soft": -1
              }
            ]
          }
        }
        ```
*   **为什么更安全？**
    *   **权限不外泄**：容器进程（PID 1）在“出生”时就已经继承了父进程（containerd handler）设定好的 unlimited 限制，因此**容器本身不需要 `CAP_SYS_RESOURCE` 特权**。
    *   **精准隔离**：只有明确声明使用该 `RuntimeClass` 的 Pod 才会放开限制。普通 Pod 依然受到默认 handler 的 64KB 硬限制保护。
*   **可行性评估**：
    *   **结论**：**高可行性**。它是 K8s 官方推荐的用于处理特殊硬件需求或安全沙箱需求的标准方案。
    *   **前提条件**：需要平台管理员具备对物理节点容器运行时配置的控制权（即能够执行一次性的节点初始化配置）。
