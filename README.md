# crater-insights
对于 [RAIDS Lab Crater](https://github.com/raids-lab/crater) 项目的个人理解与洞见。

## 项目结构

```
crater-insights/
├── README.md                          # 项目说明文档
├── prompt.md                          # 文档编写规范和提示词
│
├── general/                           # 通用架构与设计文档
│   ├── backend.md                     # 后端项目结构、路由注册机制、Manager 接口设计
│   ├── frontend.md                    # 前端项目结构、组件架构分层、路由系统、页面布局组件
│   ├── namespace&deployment.md        # Kubernetes 命名空间和部署组件分析、Pod 类型判断逻辑
│   ├── architecture.md                # [TODO] 整体架构文档
│   └── storage.md                     # [TODO] 文件系统相关文档
│
├── infrastructure/                    # 基础设施相关文档
│   ├── ceph.md                        # [TODO] Ceph 存储系统
│   └── lxcfs-webhook.md               # [TODO] lxcfs-webhook
│
├── raids-lab/                         # 与 RAIDS Lab 实验室强耦合的内容
│   └── ldap.md                        # [TODO] ACT LDAP 认证流程
│
├── job-management/                    # 作业管理相关文档
│   └── scheduling.md                  # [TODO] 不同账户下作业的排队机制和 volcano 的 queue 与调度
│
├── resource-management/               # 资源管理相关文档
│   ├── resource-binding.md            # [TODO] 绑核和 NUMA 亲和性
│   └── autoscaling.md                 # [TODO] 动态扩缩容
│
└── resources-monitoring/              # 资源监控相关文档
    └── node-load.md                   # 节点负载功能：Pod 查询优化、API 接口分层、前端表格展示
```
