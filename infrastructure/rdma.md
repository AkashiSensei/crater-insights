# RDMA

RDMA 是 Crater 系统的扩展功能之一。

本文档记录系统如何配置 k8s 容器使用 RDMA 功能，以及其相关的基本组件工作原理。

## 更新记录

> **2026-03-05** | [docs: rdma pod tolerance (#358)](https://github.com/raids-lab/crater/commit/4f3167be98c8148d8dd67be4125ecc920cc7c8d7) | `4f3167b`
> 本 RDMA 文档新增「RDMA 资源发现与上报全流程」与「故障排查」两节。「资源发现与上报全流程」给出从硬件层到平台同步层的六步流程概览，并详解三个核心组件：NVIDIA Network Operator（严格状态机、mofed.wait 锁与唯一解锁条件）、MOFED 驱动 Pod（驱动搬运与安装、持久性与 DKMS 自愈陷阱）、RDMA Shared Device Plugin Pod（Wait 机制与污点容忍），以及关键配置查询表。「故障排查」按硬件与驱动层、InfiniBand 运行状态、K8s 资源上报（Device Plugin）、驱动自愈与 K8s 状态脱节（DKMS 案例）四步提供现场排查命令与案例分析。相关现象与背景可参考 [Issue #355](https://github.com/raids-lab/crater/issues/355)。

> **2026-03-01** | [docs: rdma ulimit (#353)](https://github.com/raids-lab/crater/commit/7014fc16188eb8cbbc28fcd399c0401e9126383e) | `7014fc1`
> 重新梳理了容器内 RDMA 内存锁定的解除方案，将实践重点从高风险的容器提权转向更安全的运行时配置。文档补充了 `containerd` 的 `base_runtime_spec` 完整配置机理，详细阐述了 `CAP_IPC_LOCK`（权限准入）与 `RLIMIT_MEMLOCK`（配额限制）的分层协作原理，并提供了基于 `RuntimeClass` 实现资源限制精准隔离的生产级实践指南。

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
    1.  **标记类型**：首先需要将同步上来的资源手动修改类型。例如，将 `nvidia.com/v100`标记为 `gpu` 类型，将 `rdma/rdma_v100` 标记为 `rdma` 类型。
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
RDMA 的核心是让 hardware（网卡 HCA）绕过 CPU 直接访问远程内存。
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

### 方案 A：节点运行时全局配置 (Global Modification)
在运行 RDMA 作业的物理节点上，修改容器运行时的默认资源限制蓝图。

*   **操作要点**：
    1.  **宿主机提权 (必须)**：执行 `EDITOR=vim systemctl edit containerd`，在 `[Service]` 段落添加 `LimitMEMLOCK=infinity`。这确保了父进程拥有分发额度的内核权限。
    2.  **生成完整规范模板 (关键)**：`containerd` 不支持类似 Docker 的 `default_ulimits` 散装配置。必须先生成一个完整的 OCI 规范文件作为模板：
        ```bash
        ctr oci spec > /etc/containerd/cri-base.json
        ```
    3.  **修改模板限制**：编辑 `/etc/containerd/cri-base.json`，在 `process.rlimits` 数组中找到 `type: "RLIMIT_MEMLOCK"`，将其 `hard` 和 `soft` 修改为 `18446744073709551615` (表示无限)。
    4.  **在配置中应用**：在 `/etc/containerd/config.toml` 的对应运行时（如 `nvidia`）下配置：
        ```toml
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          base_runtime_spec = "/etc/containerd/cri-base.json"
        ```
*   **官方支持证据**：
    `containerd` 官方文档明确指出 `base_runtime_spec` 必须指向一个通过 `ctr oci spec` 生成的完整 JSON 文件。
    *   **官方配置参考**：[containerd CRI Configuration - base_runtime_spec](https://github.com/containerd/containerd/blob/main/docs/cri/config.md?plain=1#L358)
*   **常见报错排查**：
    *   **现象**：`unable to restrict sys entries without a private MNT namespace`
    *   **根源**：提供的 JSON 文件不完整（仅包含 `rlimits`）。`base_runtime_spec` 会**替换**而非**合并**默认配置，必须提供包含 `namespaces` 等完整定义的 JSON。

### 方案 B：增加容器特权（不推荐，安全性差）
在 K8s Pod Spec 的 `securityContext` 中额外添加 **`CAP_SYS_RESOURCE`** 权限。
*   **效果**：赋予容器内的 root 用户调高硬限制的能力。
*   **操作**：需在容器 Entrypoint 中手动执行 `ulimit -l unlimited`。
*   **风险分析（越界行为）**：
    *   **突破系统配额**：`CAP_SYS_RESOURCE` 是 Linux 内核中最危险的特权之一。获得该权限的用户不仅可以修改 `memlock`，还可以随意调高**文件描述符上限 (nofile)**、**进程数上限 (nproc)**、甚至绕过磁盘配额检查。
    *   **拒绝服务攻击 (DoS)**：恶意用户可以利用此权限耗尽系统的所有文件句柄或进程号，导致宿主机及节点上其他容器因无法打开新文件或创建新进程而崩溃。
    *   **破坏隔离性**：这严重违反了容器的最小权限原则，使得容器不再是一个受限的安全沙箱。

### 方案 C：使用 RuntimeClass (精准修改/隔离方案)
这是在安全性与灵活性之间取得最佳平衡的推荐方案。其核心思想是：**由具备特权的“管家”（运行时）提前为特定容器破开天花板，而保持其他容器的限制不变。**

*   **当前环境限制说明**：
    目前许多集群的 `containerd` 配置中设置了 `default_runtime_name = "nvidia"`。这意味着所有 Pod 默认都走同一个运行时。若直接修改 `nvidia` 运行时的配置，效果将退化为方案 A（全局放开）。
*   **精准隔离的实现机理**：
    1.  **节点侧新增 Handler**：在 `containerd` 配置文件中保留默认的 `nvidia` 不变，新增一个专门的 `rdma` handler（或叫 `nvidia-rdma`）。
    2.  **配置注入 (避坑指南)**：
        *   **关键点：关于 `base_runtime_spec` 的 JSON 陷阱**。必须使用 `ctr oci spec` 生成并修改后的 **100% 完整** JSON 文件。如果 JSON 仅包含 `rlimits` 片段，`containerd` 将会用该片段**覆盖（而不是合并）**默认配置，导致容器丢失命名空间、挂载点等核心基因，引发报错：`unable to restrict sys entries without a private MNT namespace`。
    3.  **集群侧定义资源**：在 K8s 中创建一个 `RuntimeClass` 对象（如 `name: rdma-runtime`），其 `handler` 字段指向刚才新增的名称。
    4.  **平台精准注入**：Crater 平台仅在检测到作业明确启用了 RDMA 时，才在 Pod Spec 中注入 `runtimeClassName: rdma-runtime`。
*   **配置示例**：
    *   **/etc/containerd/config.toml** (新增 handler):
        ```toml
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.rdma]
          runtime_type = "io.containerd.runc.v2"
          base_runtime_spec = "/etc/containerd/rdma-base.json"
        ```
*   **为什么更安全？**
    *   **权限不外泄**：容器进程在“出生”时即继承了 unlimited 限制，**无需 `CAP_SYS_RESOURCE` 特权**。
    *   **严格隔离**：未声明该 `RuntimeClass` 的普通 Pod 依然受到 64KB 硬限制保护。
*   **可行性评估**：
    *   **结论**：**高可行性/生产级推荐**。

### 最终实现

最终选用方案 A 进行实现，全局修改了节点 `dell-gpu-06` 和 `dell-gpu-32` 上的配置。

同时解除了 contaienrd 自身的限制：

```
[Service]
LimitMEMLOCK=infinity
```

但实际上没必要，通过权限检查，containerd 进程本身有突破这个限制的权限，即 `CAP_SYS_RESOURCE`。

## RDMA 资源发现与上报全流程

Crater 平台实现从底层物理网卡到 K8s 扩展资源（如 `rdma/rdma_v100`）的自动化发现与上报，涉及硬件、驱动管理程序及多个 K8s 组件的协同工作。

### 流程概览

1. **硬件层**：宿主机物理安装 Mellanox 系列网卡。
2. **编排层**：**NVIDIA Network Operator** 识别节点需求，下发管理策略。
3. **驱动加载层**：**MOFED 驱动 Pod** 在宿主机内核中安装并加载 RDMA 相关的内核模块。
4. **状态同步**：驱动加载成功后，Operator 自动更新节点标签，释放“等待”状态（`mofed.wait` 变为 `false`）。
5. **资源发现层**：**RDMA Shared Device Plugin Pod** 被调度到节点，扫描物理网卡并向 Kubelet 注册扩展资源。
6. **平台同步层**：Crater 后端通过 `SyncResource` 接口将 K8s 资源同步到数据库并完成 GPU-RDMA 的拓扑绑定。

### 核心组件详解

#### 1. NVIDIA Network Operator
*   **本质定义**：它是集群中 RDMA 环境的“总管家”，是一个 Kubernetes Operator 模式的控制器。
*   **核心功用**：负责管理所有网络相关基础设施的生命周期。它不直接传输数据，而是通过监听集群状态，自动化地在各个节点上部署驱动、插件和配置（如 `NicClusterPolicy`），确保集群的网络能力与声明的策略一致。
*   **工作逻辑 (Strict Status Machine)**：Operator 的逻辑极其“严谨且死板”：
    *   **状态回滚**：一旦检测到节点的内核版本（`kernel-version.full` 标签）发生变化，它会立即认为原有的驱动环境已不可信。
    *   **逻辑锁死**：它会自动将节点标签重置为 `mofed.wait=true`，这会直接导致下游所有依赖此标签的插件（如 `rdma-shared-dp`）被停止或拒绝调度。
    *   **唯一解锁信号**：它只信任由它**亲手调度**并成功运行的 `mofed-driver` Pod 上报的成功信号。即便宿主机通过 DKMS 等机制已经在物理层面完成了驱动自愈，只要该 Pod 没能运行（如被污点卡住），Operator 就会永远处于“逻辑锁死”状态。

#### 2. MOFED 驱动 Pod (mofed-driver)
*   **本质定义**：它是一个具有特权（Privileged）的容器，其内部打包了 Mellanox OFED 驱动的安装包和工具。
*   **核心功用**：**它不是驱动本身，而是驱动的“搬运工”和“安装工”**。当该 Pod 在节点运行时，它会将网卡运行所需的内核模块（如 `mlx5_core`, `mlx5_ib`）编译并加载到**宿主机内核**中。
*   **驱动持久性与自愈机制说明**：
    *   **持久性**：驱动模块一旦加载到内核，即使 Pod 停止运行或被删除，只要宿主机不重启，驱动依然会保持工作状态。
    *   **DKMS 自愈 (关键陷阱)**：如果宿主机安装了驱动的 DKMS (Dynamic Kernel Module Support) 版本，在内核升级后，系统会自动触发 DKMS 重新编译并加载驱动（可通过 `dkms status` 查看，如 `mlnx-ofed-kernel` 处于 `installed` 状态）。
    *   **后果**：这会导致“物理驱动已就绪”但“K8s 编排层仍认为失效”的矛盾状态。由于 `mofed-driver` Pod 因为污点等原因没能运行，Operator 无法接收到成功信号，从而锁死后续的资源发现流程。
*   **关键约束**：其 `nodeSelector` 必须严格匹配宿主机的内核版本。如果宿主机内核升级后没有对应版本的 Pod 运行，宿主机将丢失 RDMA 能力。

#### 3. RDMA Shared Device Plugin Pod (rdma-shared-dp)
*   **本质定义**：即 **RDMA 共享设备插件**，是 K8s 设备插件框架（Device Plugin Framework）的标准实现。
*   **核心功用**：它负责“搭桥”。它运行在用户态，通过扫描宿主机 `/dev/infiniband/` 下的字符设备，统计可用网卡数量，并与 Kubelet 通信，将这些硬件声明为 K8s 里的虚拟资源（如 `rdma/rdma_v100`）。
*   **关键配置**：
    *   **Wait 机制**：它依赖 `network.nvidia.com/operator.mofed.wait: "false"` 标签。
        *   *触发逻辑*：当 Operator 检测到节点加入、**内核升级**或驱动配置变更时，会自动将此标签设为 `true`。
        *   *释放逻辑*：只有当 `mofed-driver` Pod 成功运行并确认驱动就绪后，Operator 才会将其改为 `false`。
    *   **污点容忍 (Tolerations)**：在 Crater 的独占节点环境中，该 Pod 必须配置容忍 `crater.raids.io/account` 污点，否则会因无法调度而导致该节点即便硬件正常也无法上报 RDMA 资源。

### 关键配置查询指南

| 查询目标 | 命令 / 路径 | 关键字段 / 预期结果 |
| :--- | :--- | :--- |
| **Operator 状态** | `kubectl get pods -n nvidia-network-operator` | `network-operator-xxx` 处于 Running |
| **驱动安装 Pod** | `kubectl get ds -n nvidia-network-operator \| grep mofed` | 确认对应内核版本的 DaemonSet 存在 |
| **宿主机驱动状态** | `lsmod \| grep mlx5_ib` | 模块已成功加载到内核 |
| **调度等待标签** | `kubectl get node <node> --show-labels` | `mofed.wait` 必须为 `false` |
| **插件 Pod 状态** | `kubectl get pods -n nvidia-network-operator -o wide` | `rdma-shared-dp-xxx` 在目标节点正常运行 |
| **K8s 资源可见性** | `kubectl describe node <node>` | `Allocatable` 中出现 `rdma/rdma_v100` |

---

## 故障排查 (Troubleshooting)

当遇到 RDMA 无法正常调度或使用时，可按以下步骤进行 SSH 现场排查。

### 1. 硬件与驱动层检查

首先确认 Mellanox (RDMA) 驱动是否正常加载。

```bash
# 检查网卡硬件是否在 PCI 总线上（确认网卡没“掉”）
lspci -vnn | grep -i mellanox

# 检查内核模块是否已加载
lsmod | grep mlx5_ib

# 检查 dmesg 日志，看是否有驱动加载失败的报错
# 重点关注: "failed", "error", "command failed"
sudo dmesg | grep -E "mlx5|ib_|rdma" | tail -n 50
```

### 2. InfiniBand 运行状态检查

如果驱动加载正常，接下来检查网络协议栈的状态。

```bash
# 检查 IB 设备状态
# 预期：State 为 ACTIVE, Physical state 为 LinkUp
ibstat

# 检查用户态 Verbs 库是否能看到设备
# 预期：显示 hca_id (如 mlx5_0) 及其端口信息
ibv_devinfo

# 检查设备文件是否存在
# 预期：能看到 /dev/infiniband/uverbsX 等文件
ls -l /dev/infiniband/
```

### 3. Kubernetes 资源上报检查 (Device Plugin)

如果硬件和驱动都正常，但 K8s 里资源依然是 0，说明上报插件 (Device Plugin) 出了问题。

```bash
# 找到在该节点上运行的 RDMA 插件 Pod
# 通常在 kube-system 或 nvidia-gpu-operator 命名空间
kubectl get pods -A -o wide | grep $(hostname) | grep -E "rdma|device-plugin"

# 查看插件日志 (检查是否有 "no devices found" 或 "error registering")
kubectl logs -n <namespace> <pod_name>

### 4. 驱动自愈与 K8s 状态脱节 (DKMS Case)

如果 `ibstat` 正常但 K8s 资源为 0，且 `mofed.wait=true`，请检查宿主机 DKMS 状态。

```bash
# 在宿主机执行
dkms status
```

**案例分析**：
如果你看到类似下方的输出，说明驱动已通过系统 DKMS 自行安装，无需 `mofed-driver` Pod 介入安装：
```text
mlnx-ofed-kernel/25.10.OFED.25.10.1.7.1.1, 5.15.0-170-generic, x86_64: installed
nvidia/580.126.16, 5.15.0-170-generic, x86_64: installed
```
