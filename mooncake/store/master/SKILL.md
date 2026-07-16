# Mooncake Store — Master 元数据主控优化

Master Service 是 Mooncake Store 的中央协调器，负责元数据管理、segment 分配、租约 (lease) 管理和驱逐决策。其高可用性和一致性直接影响整个 Store 的可靠性。

## 源代码地图

| 文件 | 说明 |
|------|------|
| `mooncake-store/src/mooncake_master/master_service.cpp` | Master 主逻辑入口 |
| `mooncake-store/src/mooncake_master/segment_manager.cpp` | Segment 分配与回收 |
| `mooncake-store/src/mooncake_master/eviction_manager.cpp` | 驱逐管理器 (TTL + 容量) |
| `mooncake-store/src/mooncake_master/lease_manager.cpp` | 租约管理 (hard 5s / soft 30min) |
| `mooncake-store/src/mooncake_master/replica_manager.cpp` | 副本管理 |
| `mooncake-store/src/mooncake_master/metadata_store.cpp` | 元数据持久化 (HTTP/ETCD/Redis) |

## 优化维度

### 1. 高可用与故障转移
- **代码入口**: `master_service.cpp`, `metadata_store.cpp`
- **当前状态**: 依赖外部 metadata backend (ETCD/Redis) 提供持久化，Master 本身可能单点
- **关键问题**:
  - Master 是单点还是多副本？故障转移时间 (failover time)？
  - 如果 Master 挂了，已分配的 segment 和 lease 是否仍然有效？
  - Leader 选举机制？（依赖 ETCD 的 lease？自实现 Raft？）
  - Split-brain 防护？
- **Domain Knowledge 映射**:
  - `algorithms/distributed-consensus/KNOWLEDGE.md` — Raft, leader 选举
  - `architecture/cloud-native/KNOWLEDGE.md` — 云原生 HA 模式
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 故障恢复

### 2. Segment 分配策略
- **代码入口**: `segment_manager.cpp`
- **当前状态**: 1024 分片，Master 集中分配
- **关键问题**:
  - 分配算法的复杂度？大规模集群下是否有瓶颈？
  - 是否考虑了存储节点的异构性？（不同节点的 DRAM/SSD 容量差异）
  - Segment 的预分配和热预留？
  - 分配失败时的重试和退避？
- **Domain Knowledge 映射**:
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 资源分配算法
  - `architecture/cloud-native/KNOWLEDGE.md` — 分片策略

### 3. 驱逐策略 (Eviction)
- **代码入口**: `eviction_manager.cpp`
- **当前状态**: 基于 TTL (hard 5s, soft 30min) + 容量驱逐
- **关键问题**:
  - 仅 TTL 是否足够？LLM 推理中 KV cache 的访问有明显的时间局部性（同一对话）和前缀局部性
  - 是否考虑 KV cache 的 "价值"？（前缀层被命中概率更高，应更不容易被驱逐）
  - 驱逐粒度？（整对象？还是可以部分驱逐？）
  - 驱逐的写放大？（驱逐时是否需要回写？）
- **Domain Knowledge 映射**:
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 缓存替换算法 (LRU/LFU/ARC/LIRS)
  - `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 层次化驱逐

### 4. 租约 (Lease) 与 TTL 管理
- **代码入口**: `lease_manager.cpp`
- **当前状态**: hard TTL 5s, soft-pin 30min
- **关键问题**:
  - 5s hard TTL 的选择依据？如果 decode 超过 5s 会怎样？
  - 租约续期的开销？（高频续期对 Master 的压力）
  - 客户端崩溃后的租约清理？
  - 是否支持客户端主动释放租约（提前归还）？
- **Domain Knowledge 映射**:
  - `architecture/distributed-systems/KNOWLEDGE.md` — lease 设计模式
  - `algorithms/distributed-consensus/KNOWLEDGE.md` — 租约与一致性

### 5. 元数据后端 (Metadata Backend)
- **代码入口**: `metadata_store.cpp`
- **当前状态**: 支持 HTTP / ETCD / Redis 三种后端
- **关键问题**:
  - 各后端的延迟特征？（ETCD 的共识开销 vs. Redis 的单点风险）
  - 元数据的缓存策略？（client-side caching 减少后端压力）
  - 元数据的大小和增长趋势？
- **Domain Knowledge 映射**:
  - `architecture/cloud-native/KNOWLEDGE.md` — 元数据存储选型
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 后端运维

### 6. Master 可观测性
- **关键指标**:
  - Segment 利用率、分配/释放速率
  - 租约过期/续期速率
  - 驱逐速率和驱逐延迟
  - Master 请求延迟 (p50/p99) 和 QPS
  - Metadata backend 的延迟和错误率
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
