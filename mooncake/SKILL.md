# Mooncake Optimization Router

Mooncake 项目的优化问题路由器。根据用户描述的优化问题关键词，匹配到对应的 Mooncake 组件，并路由到组件级优化 skill。

## ⚠️ 前置检查：目标仓库路径

在开始分析前，确认 Mooncake 源代码所在路径。按以下优先级确定：

1. 环境变量 `MOONCAKE_REPO`
2. 默认路径 `~/src/Mooncake`

如果路径不存在：
- 告知用户并给出克隆建议：`git clone https://github.com/kvcache-ai/Mooncake.git ~/src/Mooncake`
- 如果用户选择先不克隆，则仅基于公开文档和论文洞察生成方案（标注"未验证源代码"）

## Mooncake 架构速览

Mooncake 是一个 KVCache 中心化的 LLM 推理服务平台，采用三层架构：

```
┌──────────────────────────────────────────────────────┐
│                 Application Layer                     │
│  vLLM / SGLang / TensorRT-LLM / LMDeploy / LMCache  │
│              mooncake-integration/                    │
├──────────────────────────────────────────────────────┤
│              Storage & Control Plane                  │
│    Mooncake Store (Master + Client + P2P Store)      │
│              mooncake-store/                          │
│    Conductor (KV Cache Indexer + Prefix Routing)     │
│              docs/design/conductor/                   │
├──────────────────────────────────────────────────────┤
│                  Data Plane                           │
│    Transfer Engine / TENT (Multi-Protocol Transport) │
│    RDMA · TCP · NVLink · EFA · CXL · GPUDirect      │
│              mooncake-transfer-engine/                │
└──────────────────────────────────────────────────────┘
```

## 组件路由表

| 组件 | 路由目录 | 关键职责 | 匹配关键词 |
|------|---------|---------|-----------|
| **Transfer Engine / TENT** | `transfer-engine/` | 多协议数据平面、P2P 传输、内存注册 | transfer, transport, RDMA, TCP, NVLink, EFA, CXL, latency, bandwidth, 传输, 延迟, 带宽, batch, slice, topology, 拓扑, memory registration, 内存注册, GPUDirect, TENT, 多路径, multi-path, congestion, 拥塞 |
| **Mooncake Store** | `store/` | 分布式 KV 存储、主从元数据、分级存储 | store, KV cache, put, get, eviction, 驱逐, replica, 副本, master, metadata, 元数据, tiered storage, G1, G2, G3, segment, 分段, lease, TTL, 过期, consistency, 一致性, HA, 高可用, failover, P2P store, allocation, 分配, fragmentation, 碎片 |
| **Conductor** | `conductor/` | KV Cache 索引器、前缀感知路由 | conductor, routing, 路由, prefix cache, 前缀缓存, hash, 哈希, index, 索引, cache hit, 缓存命中, ModelContext, XXH3, event, 事件, register, query |
| **Framework Integrations** | `integrations/` | Python 绑定、框架连接器 | vLLM, SGLang, TensorRT-LLM, LMDeploy, LMCache, connector, 连接器, integration, 集成, framework, 框架, python, pybind, binding, allocator, 分配器 |
| **Build & Deploy** | `build-deploy/` | 编译构建、依赖管理、发布 | build, 构建, compile, 编译, CMake, dependency, 依赖, wheel, install, 安装, deploy, 部署, CI/CD, packaging, 打包, docker, container, 容器 |
| **Cross-Cutting Operations** | `operations/` | 监控、日志、benchmark、SRE | monitor, 监控, log, 日志, alert, 告警, SRE, benchmark, 压测, 基准测试, profile, 性能分析, debug, 调试, error, 错误, recovery, 恢复, health check, 健康检查, observability, 可观测性, tracing, metrics, 指标 |

## 路由逻辑

1. **精确匹配**：如果用户在优化问题中明确提到组件名（如 "Transfer Engine 的 RDMA 延迟"），直接路由
2. **关键词匹配**：根据上表匹配最多关键词的组件
3. **多组件匹配**：如果问题涉及多个组件（如 "Transfer Engine 和 Store 之间的交互优化"），路由到所有相关组件，合并分析结果
4. **兜底路由**：如果问题无法明确归类（如 "如何优化 Mooncake 的整体性能"），路由到所有组件，生成综合方案

## 分析流程

### 1. 路由到组件

根据关键词匹配，确定目标组件。如果有多个候选，优先匹配关键词最多的组件；如果关键词覆盖度相近，选择多个。

### 2. 读取组件 Skill

读取 `<component>/SKILL.md`，获取：
- 源代码地图（关键文件路径）
- 优化维度列表
- 每个维度的领域知识映射

### 3. 读取源代码

参照 `repo-map.md` 获取准确的源文件路径，然后：
- 使用 Grep 搜索关键函数/类定义
- 使用 Read 阅读关键源文件
- 理解当前实现

### 4. 检索领域知识

对每个优化维度：
- 从领域知识库检索相关论文
- 阅读 KNOWLEDGE.md 中的相关章节
- 提取可复用的洞察

### 5. 评估与生成

- 按 common/SKILL.md 中的评估框架判断可应用性
- 按根 SKILL.md 中的模板生成方案
- 输出到 proposals/ 并更新 KNOWLEDGE.md

## 组件间交叉分析

某些优化问题可能涉及跨组件交互。如果发现以下模式，应考虑多个组件联合分析：

| 跨组件场景 | 涉及组件 | 分析重点 |
|-----------|---------|---------|
| Store 使用 Transfer Engine 的数据路径 | store + transfer-engine | put/get 操作中的传输协议选择 |
| Conductor 与 Store 的缓存一致性 | conductor + store | 缓存索引更新与驱逐的协同 |
| Python 绑定层的序列化开销 | integrations + transfer-engine | pybind11 ↔ C++ 的数据拷贝 |
| 端到端延迟分析 | transfer-engine + store + integrations | 请求全路径追踪 |
| 部署拓扑优化 | build-deploy + operations | 编译选项 × 运行时配置的协同 |
