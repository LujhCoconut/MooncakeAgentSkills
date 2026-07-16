# Mooncake Store — Replication & Placement 优化

副本管理与数据放置是分布式存储系统的 **可靠性基础**——决定数据的冗余度、可用性和访问延迟。在 Mooncake 中，KV cache 的 immutable 特性简化了复制协议，但 LLM 推理场景对延迟的敏感度又提高了放置决策的重要性。

> **理论根基**: 参见 `../architecture.md` §2.4 (CAP 分析)、§2.5 (经典问题映射)、§4.2 (前缀缓存的复制含义)

## 分布式存储系统视角

### 复制在分布式存储中的角色

```
复制的三个目的：
1. 可用性 (Availability)：节点故障时数据仍可访问
2. 持久性 (Durability)：数据不因单点故障丢失
3. 性能 (Performance)：多副本分担读压力

Mooncake 中复制的优先级：
  性能 > 可用性 > 持久性
  (KV cache 是可重新计算的，丢失只是延迟惩罚)
```

### 复制协议选择

| 协议 | 描述 | 写延迟 | 一致性 | 适用场景 |
|------|------|--------|--------|---------|
| **Chain Replication** | Writer→Head→...→Tail, Reader 读 Tail | 中 (链式传播) | 强一致 | Ceph, Azure Storage |
| **Primary-Backup** | 写 Primary, 异步同步到 Backup | 低 (只等 Primary) | 最终一致 | 通用场景 |
| **Quorum (W+R>N)** | 写 W 个副本, 读 R 个副本 | 可调节 | 可调节 | Cassandra, DynamoDB |
| **Erasure Coding** | 计算 parity, 存储分片 | 高 (计算开销) | N/A | 冷存储 |

**Mooncake 的简化**：由于对象 immutable，所有复制协议都简化为 "写完后异步复制"。**不存在 "并发写冲突"** 这个分布式存储的核心难题。

### 放置策略的理论基础

放置策略回答两个问题：
1. **副本放在哪里？**（Placement）
2. **读请求选择哪个副本？**（Read Selection）

**放置约束**：
```
Placement = f(
    capacity,         # 节点可用容量
    load,             # 节点当前负载
    topology,         # NUMA / rack / DC
    latency_to_client # 客户端到节点的延迟
)
```

**亲和性 vs. 反亲和性**：
- **亲和性 (Affinity)**：相关数据放在一起 → 减少访问延迟
  - 例：同一 request 的 KV cache 放在同一 NUMA node
- **反亲和性 (Anti-Affinity)**：副本分散在不同故障域 → 提高可用性
  - 例：副本不在同一 rack、不在同一 PDU
  - 遵循：单 rack 故障不丢全部副本

## 源代码地图

| 文件 | 说明 | 子组件 |
|------|------|--------|
| `mooncake-store/src/mooncake_master/replica_manager.cpp` | 副本管理器 | replication |
| `mooncake-store/src/mooncake_master/segment_manager.cpp` | Segment 分配 (含放置决策) | master + replication |
| `mooncake-store/src/p2p_store.cpp` | P2P Store (临时对象共享) | client + replication |
| `mooncake-store/include/p2p_store.h` | P2P Store 接口 | client + replication |

## 优化维度

### 1. 拓扑感知的副本放置（NUMA + Rack + GPU-NIC）

**这是 replication 组件最重要的优化维度。**

**分布式系统视角**：
- 放置决策的质量 = 系统可用性 × 访问延迟
- Google 的经验：Copysets 概念——随机放置 + 大量副本 ≠ 高可用（需要限制故障域组合）
- Mooncake 的放置应该是一种 **约束满足问题 (CSP)**

**三层拓扑约束**：

```
拓扑层次 (从近到远):
  Level 0: 同一 GPU (NVLink)           → 最低延迟、最小故障域
  Level 1: 同一 NUMA node               → 低延迟、共享 L3 cache
  Level 2: 同一节点 (跨 NUMA)            → 中延迟
  Level 3: 同一 rack (同一 ToR switch)   → 低网络延迟
  Level 4: 跨 rack                      → 中网络延迟
  Level 5: 跨 DC                        → 高延迟

放置约束:
  - 性能副本 (读频繁) → Level 0-2 (同 NUMA / 同节点)
  - 可用性副本 (防止丢失) → Level 3-4 (跨 rack)
  - 跨 DC 副本 → 通常对于 KV cache 无必要 (重新计算更便宜)
```

**LLM 推理的放置特征**：
- Prefill node 产生的 KV cache 需要被 Decode node 读取
- Prefill node 和 Decode node 可能在不同物理节点上（PD 分离）
- → 放置决策应考虑 **Prefill node 和 Decode node 之间的拓扑距离**
- → 对于 PD 分离场景，优先将 KV cache 副本放在 Decode node 所在 NUMA domain

**关键问题**:
- 当前放置是否感知 NUMA topology？
- 是否感知 rack/DC？
- 是否感知 GPU-NIC 的亲和性？（副本放在 NIC 可 RDMA 直达的节点）
- 异构节点（不同 DRAM/SSD 容量）的感知？
- 新节点加入/退出时的副本自动迁移（rebalance）？

**Domain Knowledge 映射**:
- `architecture/cloud-native/KNOWLEDGE.md` — 拓扑感知、Copysets
- `algorithms/resource-scheduling/KNOWLEDGE.md` — 约束满足的放置算法
- `performance/system-tuning/KNOWLEDGE.md` — NUMA 亲和性

### 2. 复制协议优化（利用 Immutability）

**分布式系统视角**：
- Chain Replication 在 Mooncake 中的适用性：写链 Head 负责 RDMA write 到下一副本
- Immutable 对象的复制 = 「数据搬运」而非「状态同步」
- 无需 WAL、无需日志重放、无需冲突解决 → 实现可以非常高效

**复制时机选择**：

| 时机 | 优点 | 缺点 |
|------|------|------|
| **同步 (写 Primary 时同时写 Backup)** | 写完即可用所有副本读 | 写延迟 = max(所有副本写延迟) |
| **异步 (写完后后台复制)** | 写延迟不受影响 | 副本不一致窗口 |
| **Lazy (首次读时才复制)** | 最小带宽 | 首次读延迟高 |

**LLM 推理的复制建议**：
- 同步复制对 KV cache 不合理——增加 Prefill→Decode 的关键路径延迟
- 异步复制足够好——KV cache 的丢失只是重算 Prefill，不是数据丢失
- Lazy 复制可能有价值——很多 KV cache 在 TTL 内根本没有被读第二次

**关键问题**:
- 当前复制是同步还是异步？
- 复制时是否使用了 Transfer Engine 的多路径能力？（并行复制到多个副本）
- 是否支持基于访问频率的动态副本数调整？（热门 KV cache 多副本）

**Domain Knowledge 映射**:
- `algorithms/distributed-consensus/KNOWLEDGE.md` — chain replication, quorum
- `architecture/cloud-native/KNOWLEDGE.md` — 复制策略

### 3. P2P 共享的去中心化优化

- **代码入口**: `p2p_store.cpp`

**去中心化发现的设计空间**：

| 方案 | 查找复杂度 | 一致性 | 运维复杂度 |
|------|----------|--------|-----------|
| **Master-mediated** | O(1) | 强 | 低 (当前) |
| **DHT (Chord/Kademlia)** | O(log N) | 最终 | 中 |
| **Gossip** | O(1) (local view) | 最终 | 低 |
| **Consistent Hashing Ring** | O(1) | 最终 | 中 |

**Mooncake 的建议**：
- P2P Store 的数据量少（仅 checkpoint 等临时对象），Master-mediated 足够
- 如果 P2P Store 扩展到更多场景，考虑 Consistent Hashing Ring（与 Store 的 1024 分片设计一致）

**关键问题**:
- P2P peer discovery 是否依赖 Master？
- 是否支持多跳传输（relay）？
- P2P 的访问控制？（谁能读我的 P2P 数据？）

**Domain Knowledge 映射**:
- `network/os-networking/KNOWLEDGE.md` — P2P 网络、DHT
- `architecture/distributed-systems/KNOWLEDGE.md` — 去中心化设计

### 4. 副本数的动态管理

**LLM 推理视角**：
- 不是所有 KV cache 都需要同样的副本数
- 热门 system prompt 的 KV cache → 需要多副本（频繁被命中）
- 一次性 request 的 KV cache → 单副本即可
- **副本数应该与 KV cache 的 "热度" 成正比**

**动态副本调整模型**：
```
replica_count(key) = base_count + f(hit_frequency, prefix_popularity)

base_count = 1 (默认)
f(x) = 0 if x < threshold
     = 1 if x >= threshold and x < high_threshold
     = 2 if x >= high_threshold

Conductor 跟踪的 cache hit 信息可以驱动这个决策
```

**关键问题**:
- 是否支持动态副本数调整？
- 副本增加的触发条件？副本减少的触发条件？
- 副本数变更是否对客户端透明？

**Domain Knowledge 映射**:
- `architecture/cloud-native/KNOWLEDGE.md` — 弹性副本
- `algorithms/resource-scheduling/KNOWLEDGE.md` — 自适应配置

### 5. Replication 可观测性

- **关键指标**:
  - 副本数分布（平均、p50、p99、最大、最小）
  - 副本同步延迟（从 Primary 写完到最后一个 Backup 写完）
  - 副本不一致窗口的时长
  - 放置决策的延迟
  - 亲和性/反亲和性违规计数
  - P2P 共享的命中率、peer 数量、发现延迟
  - 副本修复（re-replication）的频率和耗时
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
