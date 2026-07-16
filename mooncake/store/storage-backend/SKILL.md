# Mooncake Store — Storage Backend 优化

存储后端负责 Mooncake Store 的物理存储管理：DRAM 分配、SSD/NVMe 裸盘管理、以及 G1 (GPU HBM) → G2 (CPU DRAM) → G3 (NVMe SSD) 的三级数据迁移。

## 源代码地图

| 文件 | 说明 |
|------|------|
| `mooncake-store/src/storage_backend.cpp` | 存储后端抽象 (DRAM/SSD) |
| `mooncake-store/src/tiered_storage.cpp` | 三级存储管理 (G1→G2→G3) |
| `mooncake-store/src/client_allocator.cpp` | 客户端侧内存分配器 |
| `mooncake-store/include/types.h` | SegmentInfo, ObjectState 等数据结构 |

## 优化维度

### 1. DRAM 分配与管理
- **代码入口**: `storage_backend.cpp` (DRAM 部分), `client_allocator.cpp`
- **关键问题**:
  - DRAM segment 的分配粒度？（固定大小 vs. 动态？）
  - 大对象和小对象的分配策略是否区分？
  - NUMA-aware 的 DRAM 分配？（local vs. remote node 延迟差异大）
  - Memory pin 的缓存和复用？（pin/unpin 有开销）
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — 内存管理、NUMA 优化
  - `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 内存层次

### 2. SSD/NVMe 裸盘管理
- **代码入口**: `storage_backend.cpp` (SSD 部分)
- **关键问题**:
  - 是否使用 Direct I/O 绕过 page cache？（KV cache 的访问模式不适合内核缓存）
  - NVMe 的队列深度和 CPU 亲和性配置？
  - 是否使用 SPDK 等用户态驱动减少内核开销？
  - SSD 的磨损均衡和寿命管理？
- **Domain Knowledge 映射**:
  - `performance/storage-filesystem/KNOWLEDGE.md` — 存储系统优化
  - `operations/os-performance-tuning/KNOWLEDGE.md` — 内核 I/O 调优

### 3. 三级存储 (G1/G2/G3) 数据迁移
- **代码入口**: `tiered_storage.cpp`
- **当前状态**: G1 (GPU HBM) → G2 (CPU DRAM) → G3 (NVMe SSD)
- **关键问题**:
  - 迁移触发条件？（容量阈值？访问频率？预测性？）
  - 降级 (G1→G2) vs. 升级 (G3→G2) 的策略是否对称？
  - 迁移的带宽控制？（不能影响正常读写）
  - 是否支持 GPU Direct Storage (GDS) 加速 G2↔G3？
  - 热点 KV cache 是否多 tier 同时保留副本？
- **Domain Knowledge 映射**:
  - `performance/storage-filesystem/KNOWLEDGE.md` — 分级存储
  - `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 存储层次结构
  - `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU Direct Storage

### 4. 内存分配器与碎片整理
- **代码入口**: `client_allocator.cpp`, segment 分配逻辑
- **关键问题**:
  - 分配器的并发性能？（多 client 同时分配）
  - 碎片化程度和整理（compaction）策略？
  - Segment 的回收和合并？（已被驱逐的 segment 如何归还？）
  - 分配器的 lock-free 化？
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — 内存分配器
  - `algorithms/concurrent-data-structures/KNOWLEDGE.md` — 并发分配器

### 5. 存储端可观测性
- **关键指标**:
  - 各 tier 的容量使用率、命中率
  - G1→G2 / G2→G3 迁移频率和延迟
  - 分配/释放延迟 (p50/p99)
  - 碎片率
  - SSD 的 IOPS 和带宽利用率
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md` — 存储指标

## 已知优化目标

参见 `KNOWLEDGE.md`。
