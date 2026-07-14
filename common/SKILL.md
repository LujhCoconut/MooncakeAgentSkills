# Common Optimization Methodology

跨项目、跨语言的通用优化方法论。定义优化模式目录、分析工作流、方案模板和评估框架。

## 优化模式目录

常见的优化模式，按类别组织。在分析任何代码仓库时，首先扫描是否存在以下模式的应用机会。

### 1. 数据通路优化

| 模式 | 描述 | 典型场景 | 相关领域知识 |
|------|------|---------|-------------|
| **零拷贝 (Zero-Copy)** | 消除用户态/内核态之间的数据复制 | 网络传输、存储 I/O、IPC | `performance/system-tuning/` |
| **批量处理 (Batching)** | 聚合并发操作为批量以减少系统调用和锁开销 | 网络请求、数据库写入、GPU kernel launch | `performance/gpu-ai-performance/` |
| **预取 (Prefetching)** | 提前将数据加载到更快的存储层级 | CPU cache、GPU 显存、分布式存储 | `architecture/memory-storage-hierarchy/` |
| **流水线 (Pipelining)** | 将处理分解为阶段，各阶段并行执行 | 数据处理、推理 pipeline、编译 | `performance/optimization-paradigms/` |
| **异步化 (Async/Non-blocking)** | 用异步 I/O 或事件驱动替代同步阻塞 | 网络服务、RPC、文件 I/O | `performance/concurrency/` |

### 2. 资源调度优化

| 模式 | 描述 | 典型场景 | 相关领域知识 |
|------|------|---------|-------------|
| **负载均衡 (Load Balancing)** | 在多个 worker 间均衡分配任务 | 请求分发、分布式计算、GPU 分配 | `algorithms/load-balancing/` |
| **动态调度 (Dynamic Scheduling)** | 根据实时负载调整调度策略 | 自适应批大小、弹性扩缩容 | `algorithms/resource-scheduling/` |
| **亲和性调度 (Affinity)** | 基于 NUMA/拓扑感知的任务放置 | CPU pinning、GPU-NIC 绑定 | `performance/system-tuning/` |
| **优先级调度 (Priority)** | 按 QoS 等级分配资源 | 延迟敏感 vs. 吞吐优先任务 | `algorithms/resource-scheduling/` |
| **多路径调度 (Multi-Path)** | 利用多条路径并行传输 | RDMA multi-rail、NVLink multi-path | `network/os-networking/` |

### 3. 存储与缓存优化

| 模式 | 描述 | 典型场景 | 相关领域知识 |
|------|------|---------|-------------|
| **分级存储 (Tiered Storage)** | 数据在不同存储层级间自动迁移 | GPU HBM→DRAM→SSD 三级缓存 | `performance/storage-filesystem/` |
| **驱逐策略 (Eviction)** | 智能淘汰冷数据以腾出空间 | KV cache eviction、page replacement | `algorithms/resource-scheduling/` |
| **写时复制 (COW)** | 共享只读数据，修改时复制 | 快照、fork、KV cache 共享 | `performance/system-tuning/` |
| **压缩/编码** | 用压缩减少存储占用和传输带宽 | KV cache 压缩、梯度压缩 | `performance/gpu-ai-performance/` |
| **分片 (Sharding)** | 按 key 范围分布数据到多节点 | 分布式 KV 存储、一致性哈希 | `architecture/cloud-native/` |

### 4. 并发与并行优化

| 模式 | 描述 | 典型场景 | 相关领域知识 |
|------|------|---------|-------------|
| **锁粒度优化** | 减少锁竞争（读写锁、分段锁、无锁） | 高并发数据结构、metadata 管理 | `performance/concurrency/` |
| **无锁结构 (Lock-Free)** | 用 CAS/原子操作替代互斥锁 | 队列、计数器、allocator | `algorithms/concurrent-data-structures/` |
| **RCU (Read-Copy-Update)** | 读写分离，读者无锁 | 路由表、配置热更新 | `performance/concurrency/` |
| **工作窃取 (Work-Stealing)** | 空闲线程从忙线程窃取任务 | 线程池调度、协程调度 | `algorithms/resource-scheduling/` |

### 5. 可靠性优化

| 模式 | 描述 | 典型场景 | 相关领域知识 |
|------|------|---------|-------------|
| **故障检测 (Failure Detection)** | 快速发现节点/服务异常 | 心跳、lease、gossip | `architecture/distributed-systems/` |
| **优雅降级 (Graceful Degradation)** | 部分故障时维持核心功能 | 降级读写、fallback | `architecture/reliability-engineering/` |
| **重试与退避 (Retry+Backoff)** | 指数退避 + jitter 避免惊群 | RPC 重试、连接重建 | `operations/cloud-infrastructure/` |
| **熔断 (Circuit Breaker)** | 故障隔离，避免级联失败 | 微服务、外部依赖 | `operations/cloud-infrastructure/` |
| **数据校验** | 端到端校验和检测静默损坏 | 网络传输、存储 | `performance/system-tuning/` |

## 分析工作流

对任何代码仓库执行优化分析时，遵循以下流程：

### Step 1: 理解系统
- 阅读架构文档（README、design docs）
- 绘制组件拓扑图
- 识别关键数据通路（hot path）

### Step 2: 定位代码
- 根据 repo-map 定位关键源文件
- 搜索关键函数/类定义
- 理解数据流和控制流

### Step 3: 识别瓶颈
- 从代码中识别潜在瓶颈（锁、拷贝、同步点、序列化点）
- 参考已知的性能反模式（anti-pattern）
- 如果有 profiling 数据或 benchmark，优先参考

### Step 4: 检索洞察
- 对每个瓶颈类型，检索领域知识库
- 查找类似场景的论文方案
- 记录每条洞察的适用条件

### Step 5: 评估可行性
- 判断洞察是否与当前系统的约束兼容
- 评估改动量和收益
- 对每条洞察给出可应用性评级

### Step 6: 生成方案
- 按模板输出结构化方案
- 标注优先级和依赖关系
- 给出验证方案

## 可应用性评估框架

对每条论文洞察，按以下维度评估：

| 维度 | 评估标准 |
|------|---------|
| **技术可行性** | 论文方案的核心机制能否在当前语言/框架/系统上实现？ |
| **架构兼容性** | 方案是否与现有系统架构冲突？是否需要大规模重构？ |
| **性能收益** | 在当前场景下预期收益有多大？有无量化依据？ |
| **实现代价** | 代码改动量、依赖引入、配置变更的范围 |
| **风险等级** | 引入 bug 的概率、性能回退风险、兼容性破坏 |
| **时机合适** | 当前系统阶段是否适合引入该优化？是否已存在替代方案？ |

综合评级：
- **可直接应用**：所有维度都是绿灯
- **需适配**：核心技术可行，但需要非平凡的工程适配
- **仅参考**：思路有价值，但落地条件不成熟
- **不适用**：至少一个维度红灯

## 反模式：优化陷阱

需要警惕的常见陷阱：

1. **过早优化 (Premature Optimization)**：没有 profiling 数据的优化
2. **过度工程 (Over-Engineering)**：为很少触发的场景引入复杂机制
3. **论文崇拜 (Paper Worship)**：因为论文发在顶会就认为一定好，忽略实现复杂性
4. **忽视运维成本**：优化引入新的配置参数、监控指标、故障模式
5. **假设理想环境**：论文通常在受控环境评测，真实环境可能完全不同
6. **忽略人的因素**：优化方案是否容易被团队理解和维护？

## 与 domain_knowledge_agent 的检索映射

通用优化模式 → domain_knowledge_agent 目录的对应关系：

| 优化模式类别 | domain_knowledge_agent 检索路径 |
|-------------|-------------------------------|
| 数据通路优化 | `performance/system-tuning/`, `performance/optimization-paradigms/` |
| 资源调度 | `algorithms/resource-scheduling/`, `algorithms/load-balancing/` |
| 存储与缓存 | `performance/storage-filesystem/`, `architecture/memory-storage-hierarchy/` |
| 并发与并行 | `performance/concurrency/`, `algorithms/concurrent-data-structures/` |
| 网络 | `network/os-networking/`, `performance/network-performance/` |
| GPU/AI | `performance/gpu-ai-performance/`, `architecture/accelerators/` |
| 分布式一致性 | `algorithms/distributed-consensus/`, `architecture/cloud-native/` |
| 运维与可靠性 | `operations/cloud-infrastructure/`, `operations/monitoring-observability/` |
| 安全 | `security/os-security/` |
