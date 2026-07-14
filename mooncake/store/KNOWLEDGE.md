# Mooncake Store — 已知优化目标

本文档记录 Mooncake Store 组件中已识别的优化机会。

> **最后更新**: 2026-07-14
> **已分析次数**: 0 (初始种子文件)

---

## 元数据服务高可用（初始分析）

### 当前状态
- Master 使用 HTTP/ETCD/Redis 作为 metadata 后端
- 1024 分片，单 Master 或依赖外部存储的 HA
- Issue #1035 (Store V3) 中提到计划做 key-based routing 和多租户增强

### 潜在优化方向
- Master 的多副本选主（Raft 或基于 ETCD 的 leader election）
- metadata 的分级缓存（client-side caching + TTL）
- 分段状态的乐观锁 vs. 悲观锁

### 相关领域知识
- `algorithms/distributed-consensus/KNOWLEDGE.md`
- `architecture/cloud-native/KNOWLEDGE.md`

---

## 驱逐策略增强（初始分析）

### 当前状态
- 驱逐基于 TTL（hard 5s, soft-pin 30min）+ 容量限制
- 对象 immutable after Put，不支持部分驱逐

### 潜在优化方向
- 引入访问频率感知的驱逐（LFU、ARC、LIRS）
- 支持 prefix/layer-aware 驱逐（优先保留前缀层 KV cache）
- 软驱逐（标记但不立即删除，等待读取完毕）
- 基于价值的驱逐（token 位置越靠前，保留优先级越高）

### 相关领域知识
- `algorithms/resource-scheduling/KNOWLEDGE.md`
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md`

---

## 三级存储数据迁移（初始分析）

### 当前状态
- G1 (GPU HBM) / G2 (CPU DRAM) / G3 (NVMe SSD)
- 需进一步分析迁移触发条件和迁移延迟

### 潜在优化方向
- 基于使用模式的预迁移（prefetch to faster tier）
- 迁移带宽的 QoS（不影响正常读写）
- GPU direct storage (GDS) 加速 G2↔G3 迁移
- 热点数据多副本放置在不同 tier

### 相关领域知识
- `performance/storage-filesystem/KNOWLEDGE.md`
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md`
- `performance/gpu-ai-performance/KNOWLEDGE.md`
