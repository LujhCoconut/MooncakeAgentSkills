# Mooncake Store — Replication & Placement 优化

副本管理与放置策略决定 KV cache 数据的冗余度、可用性和访问延迟。核心关注点：副本数、放置的拓扑感知（亲和性/反亲和性）、以及跨节点 P2P 共享。

## 源代码地图

| 文件 | 说明 |
|------|------|
| `mooncake-store/src/mooncake_master/replica_manager.cpp` | 副本管理器 |
| `mooncake-store/src/p2p_store.cpp` | P2P Store (临时对象共享) |
| `mooncake-store/src/mooncake_master/segment_manager.cpp` | Segment 分配 (含放置决策) |
| `mooncake-store/include/p2p_store.h` | P2P Store 接口 |

## 优化维度

### 1. 副本策略
- **代码入口**: `replica_manager.cpp`
- **当前状态**: 支持多副本
- **关键问题**:
  - 副本数如何选择？（固定配置？动态按需？）
  - 写入时全量同步还是异步复制？对延迟的影响？
  - 读写 quorum 配置？(W=all, R=1? W=1, R=1?)
  - KV cache 的 immutable 特性是否可以简化复制协议？
  - 副本修复（repair）的触发条件和开销？
- **Domain Knowledge 映射**:
  - `algorithms/distributed-consensus/KNOWLEDGE.md` — 复制协议 (chain replication, quorum)
  - `architecture/cloud-native/KNOWLEDGE.md` — 副本策略

### 2. 拓扑感知放置 (Affinity)
- **代码入口**: segment 分配逻辑, `replica_manager.cpp`
- **当前状态**: 需确认是否有拓扑感知
- **关键问题**:
  - 副本是否考虑了 NUMA 亲和性？（local NUMA node 的副本访问延迟更低）
  - 是否支持 rack/DC 级别的反亲和性？（避免同一 rack 故障导致数据丢失）
  - GPU-NIC 拓扑是否影响副本放置？（优先放置到 RDMA 可达的节点）
  - 是否考虑了存储节点的异构性？（DRAM 容量、SSD 速度）
- **Domain Knowledge 映射**:
  - `architecture/cloud-native/KNOWLEDGE.md` — 拓扑感知调度
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 约束满足的放置算法
  - `performance/system-tuning/KNOWLEDGE.md` — NUMA 亲和性

### 3. P2P Store 共享
- **代码入口**: `p2p_store.cpp`
- **当前状态**: 临时对象（如 checkpoint）的 P2P 共享
- **关键问题**:
  - P2P 共享的发现机制？（如何找到持有副本的 peer？）
  - 是否支持 DHT 或 gossip 的去中心化发现？
  - P2P 传输的路径选择？（直连 vs. 通过 Master 中转）
  - P2P 的安全和访问控制？
- **Domain Knowledge 映射**:
  - `network/os-networking/KNOWLEDGE.md` — P2P 网络
  - `architecture/distributed-systems/KNOWLEDGE.md` — 去中心化

### 4. 跨区域/跨 DC 放置
- **当前状态**: 是否支持跨 DC？
- **关键问题**:
  - 跨 DC 的延迟代价是否值得？
  - KV cache 的 locality 特征（推理请求通常路由到同一 DC）
  - Geo-replication 的一致性模型？
- **Domain Knowledge 映射**:
  - `architecture/cloud-native/KNOWLEDGE.md` — 多区域部署
  - `architecture/distributed-systems/KNOWLEDGE.md` — Geo-distribution

### 5. Replication 可观测性
- **关键指标**:
  - 副本数分布、副本同步延迟
  - 副本修复频率和耗时
  - P2P 共享命中率
  - 放置决策延迟
  - 亲和性/反亲和性违规计数
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
