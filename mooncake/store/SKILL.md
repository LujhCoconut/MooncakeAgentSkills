# Mooncake Store — 子组件路由

Mooncake Store 的入口路由器。Store 是分布式 KV 缓存对象存储层，采用 Master/Client 架构，支持三级存储。本文件根据优化问题的关键词分发到 4 个子组件。

## Store 架构速览

```
┌──────────────────────────────────────────────────────┐
│                   Mooncake Store                      │
│                                                       │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   master    │  │    client    │  │ replication  │ │
│  │ 元数据主控   │  │ 客户端数据路径 │  │ 副本与放置    │ │
│  │ HA·租约·驱逐 │  │ 读写·连接·P2P │  │ 亲和·拓扑·副本│ │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘ │
│         │                │                  │         │
│         └────────────────┼──────────────────┘         │
│                          │                            │
│               ┌──────────┴──────────┐                 │
│               │  storage-backend    │                 │
│               │  存储后端            │                 │
│               │  G1(HBM)/G2(DRAM)/  │                 │
│               │  G3(SSD) 三级存储    │                 │
│               └─────────────────────┘                 │
└──────────────────────────────────────────────────────┘
```

## 子组件路由表

| 子组件 | 目录 | 职责 | 匹配关键词 |
|--------|------|------|-----------|
| **storage-backend** | `storage-backend/` | DRAM/SSD 管理、三级存储 (G1/G2/G3)、数据迁移、内存分配、碎片整理 | dram, ssd, nvme, hbm, vram, 显存, tiered storage, 分级存储, G1, G2, G3, migration, 迁移, allocator, 分配器, fragmentation, 碎片, compaction, backend, 后端, storage engine, 存储引擎, direct i/o, 裸盘, 磁盘 |
| **master** | `master/` | 元数据主控、HA/故障转移、一致性、segment 管理、租约 (lease)、TTL、驱逐 (eviction)、metadata backend | master, 主控, metadata, 元数据, HA, 高可用, failover, 故障转移, 切换, consistency, 一致性, segment, 分段, lease, 租约, TTL, 过期, eviction, 驱逐, 淘汰, LRU, etcd, redis |
| **client** | `client/` | 客户端读写路径、RealClient/DummyClient、连接池、批量操作、P2P Store | client, 客户端, real client, dummy client, put, get, read, write, 读写, connection, 连接, pool, 连接池, batch, 批量, P2P, peer-to-peer, 点对点, rpc, proxy, 代理 |
| **replication** | `replication/` | 副本策略、亲和性/反亲和性放置、拓扑感知、跨节点/rack/DC、P2P 共享 | replica, 副本, replication, 复制, placement, 放置, affinity, 亲和性, 亲和, anti-affinity, 反亲和, topology, 拓扑, rack, 机架, cross-dc, 跨数据中心, quorum, 法定人数, p2p share, 共享 |

## 路由逻辑

1. **单关键词匹配**：在用户问题中搜索上表关键词，匹配最多的子组件
2. **多子组件**：如果问题涉及存储端到端（如 "Store 的整体性能优化"），路由到所有 4 个子组件
3. **跨组件**：如果问题涉及 Store 与 Transfer Engine 的交互（如 "put 操作的 RDMA 路径优化"），同时路由到 `store/client/` 和 `transfer-engine/transport/`

## 各子组件入口

| 子组件 | SKILL.md | 主要源码 (参见 repo-map.md) |
|--------|----------|--------------------------|
| storage-backend | `storage-backend/SKILL.md` | `storage_backend.cpp`, `tiered_storage.cpp` |
| master | `master/SKILL.md` | `mooncake_master/` 全部 (6 个 cpp) |
| client | `client/SKILL.md` | `mooncake_client/` + `p2p_store.cpp` |
| replication | `replication/SKILL.md` | `replica_manager.cpp` + 放置逻辑 |

## 分析流程

1. 根据关键词路由到子组件
2. 读取子组件 `SKILL.md` 获取优化维度和领域知识映射
3. 参照 `../repo-map.md` 定位源代码
4. 按 `../../common/SKILL.md` 中的流程执行分析
5. 生成方案到 `../../proposals/`

## 已知优化目标

各子组件的 KNOWLEDGE.md 中累积沉淀。
