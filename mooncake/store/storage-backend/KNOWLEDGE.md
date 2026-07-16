# Mooncake Store — Storage Backend 已知优化目标

> **最后更新**: 2026-07-16
> **已分析次数**: 1 (Bucket Backend SSD Offloading 多维度优化)

---

## Bucket Backend SSD Offloading 写路径 Direct I/O（2026-07-16 验证）

### 当前状态 (已验证)
- `BucketStorageBackend::OpenFile()` (storage_backend.cpp:2598-2632): O_DIRECT 仅对 `FileMode::Read` 启用
- `WriteBucket()` (storage_backend.cpp:2020-2158): io_uring 路径使用 `write_aligned` + bounce buffer，但数据仍经过 page cache
- 代码注释明确说 "write latency is not sensitive, and O_DIRECT writes require 4096-byte alignment padding"
- 实际验证：写入不使用 O_DIRECT → 数据先进入 page cache → 占用本可用于 G2 KV cache 的 DRAM

### 已验证的优化方向
- [P0] 写路径启用 O_DIRECT（配合 io_uring），消除 page cache 污染
- 方案详见: `proposals/mooncake-store-bucket-ssd-offloading-optimization-2026-07-16.md`

### 相关领域知识
- Latte(FAST'26): O_DIRECT + page cache bypass for cloud storage
- architecture.md §2.2: "page cache not only useless but harmful for KV cache SSD"

---

## Bucket 驱逐粒度优化（2026-07-16 验证）

### 当前状态 (已验证)
- 驱逐以 bucket 为单位（默认 256MB, ≤500 keys）
- `SelectEvictionCandidate()` (storage_backend.cpp:2191-2236): FIFO 返回 `buckets_.begin()`，LRU 用 lazy index repair
- `PrepareEviction()` (storage_backend.cpp:2238-2341): 整桶移除 → 桶内所有 key 一起被驱逐
- FIFO 模式下冷 bucket 中的热 key 被误杀；LRU 的 last_access 是 per-bucket 的

### 已验证的优化方向
- [P1] Bucket 热度评分 + 驱逐前 hot key 提升 (compaction-like)
- [P2] Per-object 访问计数用于桶内细粒度决策
- 方案详见: `proposals/mooncake-store-bucket-ssd-offloading-optimization-2026-07-16.md`

### 相关领域知识
- FORGE(OSDI'26): group-level sync amortization
- TapeOBS(FAST'26): lifecycle-based grouping for zero-cost GC

---

## 元数据 I/O 路径优化（2026-07-16 验证）

### 当前状态 (已验证)
- `StoreBucketMetadata()` (storage_backend.cpp:2509-2539): POSIX write, 不使用 io_uring
- `LoadBucketMetadata()` (storage_backend.cpp:2541-2580): PosixFile read, 不使用 io_uring
- `Init()` (storage_backend.cpp:1593-1783): 串行逐个读取和解析 `.meta` 文件
- 即使 `USE_URING` 编译开启，metadata 读写仍走 POSIX 路径

### 已验证的优化方向
- [P1] 元数据读写走 io_uring 异步化
- [P1] Init() 批量 io_uring 读取多个 metadata 文件
- 方案详见: `proposals/mooncake-store-bucket-ssd-offloading-optimization-2026-07-16.md`

---

## One-hit-wonder 过滤与 Bucket 打包优化（2026-07-16 验证）

### 当前状态 (已验证)
- `GroupOffloadingKeysByBucket()` (storage_backend.cpp:1895-1986): 贪心 first-fit 填桶，按 size+count 限制
- 存在 `ungrouped_offloading_objects_` 机制处理跨批次碎片
- 无 offload 前的价值判断——所有被标记为 offload 的对象都写入 SSD
- 无 TTL 亲和性考虑——不同生命周期的对象可能混在同一 bucket

### 已验证的优化方向
- [P2] TTL 亲和分桶 + first-fit-decreasing bin-packing
- [P2] Delayed offload 观察窗口（DRAM LRU → 窗口过期才真正 offload）
- 方案详见: `proposals/mooncake-store-bucket-ssd-offloading-optimization-2026-07-16.md`

### 相关领域知识
- Latte(FAST'26): S3-FIFO candidate queue (72% one-hit-wonder)
- TapeOBS(FAST'26): lifecycle-based grouping

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

## SSD/NVMe 裸盘 I/O 优化（已分析，见上方 Bucket Backend 章节）

### 已验证结论
- io_uring 已在读路径使用（O_DIRECT），本地 diff 增加 batch_read + file handle cache
- 写路径不使用 O_DIRECT → 见 [P0] 优化
- SPDK 可行性需进一步评估

---

## 三级存储数据迁移（初始分析）

### 当前状态
- `tiered_storage.cpp` 不存在（文件未找到）
- G1↔G2↔G3 迁移逻辑可能在其他文件中或尚未实现独立模块
- Promotion 路径见于 `file_storage.cpp:ProcessPromotionTasks()` (SSD→DRAM)

### 潜在优化方向
- 基于访问模式的预测性预迁移（prefetch to faster tier）
- 迁移带宽 QoS（不阻塞正常读写）
- GPU Direct Storage (GDS) 加速 GPU↔SSD 直传
- 热点数据多 tier 冗余（热门 prefix 的 KV cache 同时保留在 G1 和 G2）

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
