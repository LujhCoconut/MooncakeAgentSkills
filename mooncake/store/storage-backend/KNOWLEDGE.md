# Mooncake Store — Storage Backend 已知优化目标

> **最后更新**: 2026-07-16
> **已分析次数**: 0

---

## DRAM 分配与 NUMA 感知（初始分析）

### 当前状态
- DRAM segment 通过 `storage_backend.cpp` 管理
- 需验证：分配是否感知 NUMA topology

### 潜在优化方向
- NUMA-local 优先分配（减少跨 NUMA node 的内存访问延迟）
- Segment 大小自适应（小对象用小 segment，大对象用大 segment）
- Memory pin 缓存池（避免频繁 pin/unpin 开销）
- 预分配和热预留（减少分配延迟抖动）

### 相关领域知识
- `performance/system-tuning/KNOWLEDGE.md` — NUMA 优化、内存管理
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md`

---

## SSD/NVMe 裸盘 I/O 优化（初始分析）

### 当前状态
- SSD backend 在 `storage_backend.cpp` 中
- 需验证：是否使用 Direct I/O、io_uring 或 SPDK

### 潜在优化方向
- Direct I/O 绕过 page cache（KV cache 无复用价值，page cache 浪费 DRAM）
- io_uring 替代 libaio（更低开销的异步 I/O）
- SPDK 用户态 NVMe 驱动（消除内核态上下文切换）
- NVMe 多队列与 CPU 亲和性绑定
- SSD 写入放大优化（对齐写入、批量写入）

### 相关领域知识
- `performance/storage-filesystem/KNOWLEDGE.md`
- `operations/os-performance-tuning/KNOWLEDGE.md`

---

## 三级存储数据迁移（初始分析）

### 当前状态
- `tiered_storage.cpp` 管理 G1 (GPU HBM) → G2 (CPU DRAM) → G3 (NVMe SSD)
- 需验证迁移触发条件和带宽控制

### 潜在优化方向
- 基于访问模式的预测性预迁移（prefetch to faster tier）
- 迁移带宽 QoS（不阻塞正常读写）
- GPU Direct Storage (GDS) 加速 GPU↔SSD 直传
- 热点数据多 tier 冗余（热门 prefix 的 KV cache 同时保留在 G1 和 G2）
- 冷数据渐进式降级（G1→G2→G3 逐步迁移，避免一次性大 IO）

### 相关领域知识
- `performance/storage-filesystem/KNOWLEDGE.md`
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md`
- `performance/gpu-ai-performance/KNOWLEDGE.md`

---

## 内存分配器并发优化（初始分析）

### 当前状态
- `client_allocator.cpp` 负责客户端侧内存分配
- 需验证并发分配的性能特征

### 潜在优化方向
- Lock-free / wait-free 分配器（减少多 client 竞争）
- Thread-local cache (TCMalloc / jemalloc 风格)
- 分配器的 false sharing 分析
- Segment 碎片整理（compaction）策略

### 相关领域知识
- `algorithms/concurrent-data-structures/KNOWLEDGE.md`
- `performance/system-tuning/KNOWLEDGE.md`
