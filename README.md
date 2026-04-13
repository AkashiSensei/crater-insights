# crater-insights
对于 [RAIDS Lab Crater](https://github.com/raids-lab/crater) 项目的个人理解与洞见。

## 项目结构

```
crater-insights/
├── README.md                          # 项目说明文档
├── prompt.md                          # 文档编写规范和提示词
│
├── accessibility/                     # 可访问性与用户体验相关文档
│   └── error-return.md                # 错误信息返回与展示：前后端错误处理机制、业务错误码设计、错误消息处理流程
│
├── draft/                             # 初步 idea 与草案（未成文主题的可选沉淀）
│   ├── cli-idea.md                    # crater-cli 需求与设想：命令行直连、版本兼容、CLI 与 MCP/API 的权衡
│   ├── data-collect-idea.md           # 运行时数据收集：资源申请与实际占用、使用偏好等分析素材
│   └── image-analysis-idea.md         # 镜像依赖与存储层分析：FROM 依赖树、层复用展示与权限可见性
│
├── general/                           # 通用架构与设计文档
│   ├── backend.md                     # 后端项目结构、路由注册机制、Manager 接口设计
│   ├── frontend.md                    # 前端项目结构、组件架构分层、路由系统、页面布局组件
│   ├── namespace&deployment.md        # Kubernetes 命名空间和部署组件分析、Pod 类型判断逻辑
│   ├── architecture.md                # [TODO] 整体架构文档
│   └── storage.md                     # [TODO] 文件系统相关文档
│
├── image-management/                  # 镜像管理相关文档
│   └── capabilities.md                # 镜像能力：Crater 支持的镜像相关功能列表与技术洞察
│
├── infrastructure/                    # 基础设施相关文档
│   ├── ceph.md                        # [TODO] Ceph 存储系统
│   ├── lxcfs-webhook.md               # [TODO] lxcfs-webhook
│   └── rdma.md                        # RDMA 在 Crater 系统中的配置与应用、软硬件需求及平台处理流程
│
├── raids-lab/                         # 与 RAIDS Lab 实验室强耦合的内容
│   ├── authentication.md              # ACT LDAP 认证流程：两种认证方式、用户创建与状态管理、外部服务依赖关系
│   └── go-ldap.md                     # Go LDAP 库介绍：LDAP协议基础、go-ldap/ldap和go-ldap-client库对比
│
├── job-management/                    # 作业管理相关文档
│   ├── pod-configuration.md           # Pod 配置生成，包括节点亲和性等
│   └── scheduling.md                  # [TODO] 不同账户下作业的排队机制和 volcano 的 queue 与调度
│
├── resource-management/               # 资源管理相关文档
│   ├── resource-binding.md            # [TODO] 绑核和 NUMA 亲和性
│   └── autoscaling.md                 # [TODO] 动态扩缩容
│
└── resources-monitoring/              # 资源监控相关文档
    ├── accelerator-monitoring.md       # 加速卡监控机制：体系架构、多厂商兼容现状与演进方向
    └── node-load.md                   # 节点负载功能：Pod 查询优化、API 接口分层、前端表格展示
```
