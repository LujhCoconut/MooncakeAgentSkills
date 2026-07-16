# Mooncake Store — Master 元数据主控优化

Master 是 Mooncake Store 的 **控制平面**——负责元数据管理、资源分配、租约和驱逐决策。从分布式存储系统视角看，这是 Mooncake 的「协调者」。

> **理论根基**: 参见 `../architecture.md` §2.3 (分布式架构)、§2.4 (CAP 分析)、§2.5 (分布式存储经典问题)

## 分布式存储系统视角

### 控制平面在分布式存储中的角色

| 系统 | 控制平面 | 元数据存储 | 一致性机制 |
|------|---------|-----------|-----------|
| **HDFS** | NameNode | 内存 + EditLog | 单点 + Checkpoint |
| **Ceph** | MON (Monitor) | RocksDB | Paxos 多副本 |
| **GFS** | Master | 内存 + Operation Log | 单点 + Shadow Master |
| **Mooncake** | Master Service | HTTP/ETCD/Redis | 外部 metadata store |

Mooncake 的独特之处：**控制平面不参与数据路径**（Client 直接 RDMA 读写数据），这简化了 Master 的设计但也引入了一致性挑战——Master 认为对象存在，但实际数据已因节点故障丢失。

### Master 的 CAP 位置

```
在 CAP 三角中，Master 的位置取决于配置：
- HTTP backend → 偏向 AP (单点, 但可用性高)
- ETCD backend → 偏向 CP (强一致, 但延迟高)
- Redis backend → 偏向 AP (单点风险, 但延迟低)

KV cache 场景下：
- 元数据的短暂不一致 (A > C) 是可以容忍的
  → 最多导致一次 cache miss (多算一次 prefill)
  → 不会导致数据损坏 (对象 immutable)
- 但如果 Master 不可用，新的 KV cache 无法注册
  → 可用性同样重要
```

### 控制平面的核心功能与分布式系统映射

| 功能 | 分布式系统概念 | Mooncake 实现 | 关键问题 |
|------|--------------|--------------|---------|
| **元数据持久化** | WAL / EditLog | HTTP/ETCD/Redis backend | 持久化的延迟对性能的影响？ |
| **Segment 分配** | Resource Allocation | `segment_manager.cpp` | 分配算法复杂度？大规模下的瓶颈？ |
| **驱逐 (Eviction)** | GC / Cache Replacement | `eviction_manager.cpp` | TTL 是否足够？要不要频率感知？ |
| **租约 (Lease)** | Distributed Lease | `lease_manager.cpp` | 5s hard TTL 的合理性？ |
| **高可用** | Leader Election + Failover | 依赖外部 backend | 故障转移时间？对 client 的透明性？ |

## 源代码地图

| 文件 | 说明 | 子组件 |
|------|------|--------|
| `mooncake-store/src/mooncake_master/master_service.cpp` | Master 主逻辑入口 | master |
| `mooncake-store/src/mooncake_master/segment_manager.cpp` | Segment 分配与回收 | master |
| `mooncake-store/src/mooncake_master/eviction_manager.cpp` | 驱逐管理器 (TTL + 容量) | master |
| `mooncake-store/src/mooncake_master/lease_manager.cpp` | 租约管理 (hard 5s / soft 30min) | master |
| `mooncake-store/src/mooncake_master/replica_manager.cpp` | 副本管理 | replication |
| `mooncake-store/src/mooncake_master/metadata_store.cpp` | 元数据持久化 (HTTP/ETCD/Redis) | master |

## 优化维度

### 1. 高可用与故障转移（控制平面可靠性）

- **代码入口**: `master_service.cpp`, `metadata_store.cpp`

**分布式系统视角**：
- Master 的高可用方案选择反映了 **CP vs. AP** 的权衡
- 选项 A：依赖 ETCD 的 Raft → CP, 3-5 节点, 有延迟开销
- 选项 B：Master 自建 Raft → CP, 更可控但实现复杂
- 选项 C：Primary-Backup + ETCD lease → AP, 简单但可能有短暂不可用

**KV cache 特有的 HA 考量**：
- Master 短暂不可用时（秒级），已有的 KV cache 读取不受影响（数据路径独立）
- 但新的 KV cache 写入（Prefill 完成后注册）会失败 → 推理请求延迟增加
- **目标**：故障转移时间 < 已注册 KV cache 的平均 TTL（否则驱逐和分配都乱了）

**关键问题**:
- Master 故障的检测延迟？（heartbeat interval）
- 故障转移期间对 client 的影响？（透明重连？还是返回错误？）
- Split-brain 的防护？（fencing token）
- 新 Master 接管后的状态重建时间？

**Domain Knowledge 映射**:
- `algorithms/distributed-consensus/KNOWLEDGE.md` — Raft, leader election, lease
- `architecture/cloud-native/KNOWLEDGE.md` — 云原生 HA 模式
- `operations/cloud-infrastructure/KNOWLEDGE.md` — 故障恢复

### 2. Segment 分配策略（资源调度）

- **代码入口**: `segment_manager.cpp`

**分布式系统视角**：
- Segment 分配 = 分布式存储中的 **placement 决策**
- 类比 HDFS block allocation：考虑 rack awareness, disk utilization, 负载均衡
- Mooncake 的分配决策应额外考虑：NUMA topology, DRAM 容量, 节点负载

**分配策略对比**：

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **轮询 (Round-Robin)** | 简单、均衡 | 无视异构性 | 同构集群 |
| **加权 (Weighted)** | 感知容量差异 | 需要准确的权重 | 异构集群 |
| **最小负载 (Least-Loaded)** | 自适应 | 需要实时负载信息 | 负载波动大 |
| **亲和性 (Affinity)** | 降低后续访问延迟 | 可能不均衡 | NUMA 环境 |

**关键问题**:
- 分配算法的计算复杂度？大规模集群 (1000+ nodes) 的分配延迟？
- 是否支持预分配（warm pool）以减少分配延迟？
- 分配失败时的重试策略？退避？fallback？

**Domain Knowledge 映射**:
- `algorithms/resource-scheduling/KNOWLEDGE.md` — 资源分配算法, bin packing
- `architecture/cloud-native/KNOWLEDGE.md` — 分片策略

### 3. 驱逐策略 — 从 TTL 到价值感知

- **代码入口**: `eviction_manager.cpp`

**这是 Mooncake 当前设计中优化杠杆最大的点。**

**分布式存储视角**：
- 通用缓存驱逐算法：LRU、LFU、ARC、LIRS、2Q
- 当前 Mooncake 使用 pure TTL —— 最简单的驱逐策略

**LLM 推理视角下的驱逐应该是什么样？**

```
KV cache 的「价值」不是一个标量，而是一个向量：
  Value(token, layer, age, prefix_match_prob, request_rate)

其中：
  - token 位置越靠前 → 价值越高（前缀匹配概率更高）
  - 层 (layer) 越靠前 → 价值越高（前缀层被命中概率 >> 尾部层）
  - age (年龄) → 刚创建的 KV cache 价值不确定（可能很快被驱逐，也可能被频繁访问）
  - prefix_match_prob → 如果已被 Conductor 索引到，说明有相似前缀，价值更高
  - request_rate → 热门 prompt 的 KV cache 价值高
```

**理想的驱逐策略**：
```
Weight = f(token_position, layer_position, age, hit_count, prefix_popularity)

T1: TTL-based（基础，简单）
T2: TTL + Layer-aware（前缀层更长 TTL）
T3: TTL + Layer-aware + Frequency-aware（LRU/LFU 混合）
T4: TTL + Layer-aware + Frequency-aware + Prefix-popularity（最全面）
```

**关键问题**:
- 最短路径：能否实现 T2 (Layer-aware TTL)？改动量小，但能显著提升前缀命中率
- 驱逐粒度：整对象驱逐 vs. 部分驱逐（只驱逐不活跃的层）？
- 驱逐 vs. 降级的协作：驱逐前是否先尝试降级到更慢 tier？
- 驱逐的「软驱逐」（标记驱逐但延迟删除，等 reader 完成）的实现？
- Evict 事件通知 Conductor 的延迟？

**Domain Knowledge 映射**:
- `algorithms/resource-scheduling/KNOWLEDGE.md` — LRU/LFU/ARC/LIRS, 缓存替换
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 层次驱逐
- `performance/gpu-ai-performance/KNOWLEDGE.md` — KV cache 优化

### 4. 租约与 TTL 管理（Lease 设计模式）

- **代码入口**: `lease_manager.cpp`

**分布式系统视角**：
- Lease 是分布式系统中实现「有限时间内的一致性保证」的经典模式
- Hard TTL = 客户端保证在这段时间内持有资源
- Soft TTL = 服务端的驱逐宽限期

**Lease 设计的经典问题在 Mooncake 中的体现**：

| 经典问题 | Mooncake 中的体现 | 风险 |
|---------|-----------------|------|
| **时钟偏移** | hard TTL 依赖 client-server 时钟同步 | 时钟偏差可能导致过早/过晚驱逐 |
| **客户端崩溃** | 客户端持有 lease 但 crash → 资源泄露直到 TTL 过期 | 5s 的 hard TTL 限制了泄露时长 |
| **续期风暴** | 大量 client 同时续期 → Master 压力 | 需要续期批量处理或滑动窗口 |
| **TTL 选择** | 5s 的 hard TTL 是否合理？ | 太短：续期频繁；太长：资源泄露时间长 |

**LLM 推理视角**：
- Decode 阶段的延迟通常 <5s，所以 5s hard TTL 对 decode 没问题
- 但 Prefill (长 prompt) 可能超过 5s → 需要续期
- 跨对话的 KV cache 共享需要更长 TTL (soft 30min)

**关键问题**:
- 续期的批量优化？（多个 lease 的 batch renew）
- 基于 client 健康状态的动态 TTL？（健康的 client 给更长 TTL）
- 客户端优雅关闭时主动释放 lease（加速回收）的实现？
- Lease 的单调性保证？（防止旧 lease 覆盖新 lease）

**Domain Knowledge 映射**:
- `architecture/distributed-systems/KNOWLEDGE.md` — lease 设计模式
- `algorithms/distributed-consensus/KNOWLEDGE.md` — 定时器与一致性

### 5. 元数据后端选型与优化

- **代码入口**: `metadata_store.cpp`

**分布式系统视角**：

| 后端 | 一致性 | 延迟 | 可用性 | 运维复杂度 |
|------|--------|------|--------|-----------|
| **HTTP** | 弱 | 低 (1-5ms) | 单点 | 最低 |
| **ETCD** | 强 (Raft) | 中 (5-20ms) | 高 (3-5节点) | 中 |
| **Redis** | 弱 (async replication) | 低 (<1ms) | 中 (sentinel) | 中 |

**选择建议**：
- 元数据丢失的代价？→ 需要重新 Prefill，影响延迟但不丢数据
- 因此 **AP > CP**，Redis/HTTP 可能比 ETCD 更合适
- Client-side metadata cache 可以进一步降低后端压力

**关键问题**:
- 元数据的大小估算？（每个对象一条记录 × 对象数量）
- 元数据的读写比例？（读 >> 写？写 >> 读？）
- Client-side caching 的可能性？（TTL-based cache）

**Domain Knowledge 映射**:
- `architecture/cloud-native/KNOWLEDGE.md` — 元数据存储选型
- `operations/cloud-infrastructure/KNOWLEDGE.md` — 后端运维

### 6. Master 可观测性

- **关键指标**:
  - Segment 利用率、分配/释放速率（按节点、按 tier）
  - 租约数量、过期速率、续期频率
  - 驱逐速率（按原因：TTL vs. 容量）和驱逐延迟
  - Master 请求延迟 (p50/p99) 和 QPS（按操作类型）
  - Metadata backend 的延迟和错误率
  - Master 的 CPU/内存使用、Goroutine/线程数
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。


## ⚠️ 维护规则

**当本 SKILL.md 有实质性更新（新增优化维度、修改分析流程、更新路由规则、新增领域知识映射等），必须同步检查并更新 `README.md` 中对应的项目结构、组件覆盖表、使用示例或模块说明。** 保持 README 始终反映最新的 skill 能力边界。
