# Mooncake Store — Storage Backend 优化

Storage Backend 是 Mooncake 的 **存储引擎**——负责物理存储介质的分配、管理和数据在 G1/G2/G3 三级存储间的迁移。从分布式存储系统视角看，这是 Mooncake 的「存储层次」层。

> **理论根基**: 参见 `../architecture.md` §2.2 (存储层次)、§2.5 (经典问题映射)

## 分布式存储系统视角

### 存储引擎在分布式存储中的角色

在通用分布式存储系统（如 Ceph、HDFS）中，存储引擎负责：
- **空间分配**：block / chunk / extent 的分配策略
- **数据放置**：数据在存储介质上的物理位置
- **迁移与均衡**：冷热数据的层级移动
- **碎片整理**：空间回收与压缩

Mooncake 的 Storage Backend 完全对应这些职责，但有 KV cache 特有的约束：
- 数据生命周期短（秒到分钟级），碎片化速度更快
- G1 (GPU HBM) 容量极度有限（几十 GB），驱逐压力大
- 迁移不能阻塞推理的读写路径

### 存储引擎设计的核心权衡

| 权衡 | 选项 A | 选项 B | Mooncake 当前 |
|------|--------|--------|--------------|
| **分配粒度** | 固定大小 (简单、易碎片) | 可变大小 (复杂、省空间) | Segment-based (需验证) |
| **迁移触发** | 容量阈值 (被动) | 访问模式 (主动/预测) | 需验证 |
| **迁移方式** | 同步 (阻塞读写) | 异步 (一致性问题) | 需验证 |
| **空间回收** | 立即 (安全) | 延迟批量 (高效) | TTL-based |

## 源代码地图

| 文件 | 说明 | 子组件 |
|------|------|--------|
| `mooncake-store/src/storage_backend.cpp` | DRAM/SSD 存储后端抽象 | storage-backend |
| `mooncake-store/src/tiered_storage.cpp` | 三级存储迁移逻辑 (G1↔G2↔G3) | storage-backend |
| `mooncake-store/src/client_allocator.cpp` | 客户端侧内存分配器 | storage-backend |
| `mooncake-store/include/types.h` | SegmentInfo, ObjectState | 全局 |

## 优化维度

### 1. DRAM 分配与管理（存储引擎核心）

- **代码入口**: `storage_backend.cpp` (DRAM 部分), `client_allocator.cpp`

**分布式系统视角**：
- Segment 是 DRAM 分配的基本单元——类似于 Ceph 的 PG (Placement Group) 或 HDFS 的 block
- 固定大小的 segment 在负载多样时要么浪费空间（小对象），要么限制最大对象大小
- **异构性**：不同节点的 DRAM 容量可能不同（有些节点 GPU 多、DRAM 少）

**LLM 推理视角**：
- KV cache 对象的生命周期通常是秒级（一次对话），高频分配释放 → 分配器需要高并发 + 低碎片
- G2 (DRAM) 中存储的是「温」KV cache——前缀匹配概率中等，不应与「冷」SSD 数据同等对待
- DRAM 分配应考虑 **对象大小分布**（不同模型、不同序列长度产生不同大小的 KV cache）

**关键问题**:
- Segment 大小的选择依据？不同 segment 大小适应不同对象大小的策略？
- 分配器的并发模型？多 client 同时分配时的锁竞争？
- NUMA 感知？分配在请求方所在的 NUMA node 上可以显著降低延迟
- 大页 (huge page) 的使用？降低 TLB miss，提升 pin 效率

**Domain Knowledge 映射**:
- `performance/system-tuning/KNOWLEDGE.md` — 内存管理、NUMA、huge pages
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 层次化内存设计
- `algorithms/concurrent-data-structures/KNOWLEDGE.md` — 并发分配器

### 2. SSD/NVMe 裸盘管理（冷存储层）

- **代码入口**: `storage_backend.cpp` (SSD 部分)

**分布式系统视角**：
- G3 (NVMe SSD) 在 Mooncake 中的角色类似于 HDFS 的磁盘层——存「冷」数据
- 但与 HDFS 不同：KV cache 在 SSD 上的数据几乎没有复用价值（超过软 TTL 的 KV cache 几乎不会被再次访问）
- 因此 **page cache 不仅无用，还有害**（占用本来可以分配给 G2 的 DRAM）

**I/O 路径分析**：
```
应用 (Client)
  → storage_backend.cpp → write()
    → 是否经过 page cache? 如果用了 buffered I/O，数据先到 page cache
    → page cache 占用 DRAM (本可用于 G2!)
    → 然后由内核异步刷到 SSD
```

**结论**：G3 应该使用 Direct I/O (O_DIRECT) 或 io_uring，绕过 page cache

**关键问题**:
- 是否使用 Direct I/O？是否有 page cache 污染 DRAM 的问题？
- NVMe 多队列的 CPU 亲和性配置？（每个 CPU core 绑定一个 NVMe queue pair）
- io_uring vs. libaio？io_uring 有更低的系统调用开销
- 写入放大？KV cache 对象的写入是否对齐到 SSD block 大小？
- SPDK (用户态 NVMe 驱动) 的可行性？消除内核态上下文切换

**Domain Knowledge 映射**:
- `performance/storage-filesystem/KNOWLEDGE.md` — LFS, 裸盘 I/O, 写入放大
- `operations/os-performance-tuning/KNOWLEDGE.md` — io_uring, Direct I/O, 内核旁路

### 3. 三级存储数据迁移（层次化管理）

- **代码入口**: `tiered_storage.cpp`

**这是存储引擎的核心决策点**——如何在三级存储间移动数据：

```
┌──────────────────────────────────────────────────────────┐
│ 迁移策略的核心是回答三个问题：                              │
│                                                          │
│ 1. 什么时候迁移？（触发条件）                               │
│    ├── G1→G2: 容量压力？LRU？TTL 过期？                    │
│    ├── G2→G3: 容量压力？TTL 过期？冷检测？                  │
│    ├── G2→G1: 访问频率上升？预取？                          │
│    └── G3→G2: 缓存命中前的预取？                            │
│                                                          │
│ 2. 迁移多少？（粒度）                                       │
│    ├── 整对象迁移（简单，大对象可能阻塞）                      │
│    └── 部分迁移（复杂，但 KV cache 按层组织的特性支持分层迁移） │
│                                                          │
│ 3. 迁移速度？（带宽控制）                                    │
│    ├── 全速迁移（阻塞正常读写）                              │
│    └── 限速迁移（需要 QoS 机制）                             │
└──────────────────────────────────────────────────────────┘
```

**LLM 推理特有的迁移洞察**：
- KV cache 的「价值」随 token 位置衰减。前缀层（layer 0-5）被命中概率远高于尾部层
- 迁移策略可以 **层感知**：优先驱逐尾部层的 KV cache，优先保留前缀层在更快 tier
- GPU Direct Storage (GDS) 可实现 GPU↔SSD 直传，跳过 CPU DRAM，降低 G1→G3 的迁移成本

**关键问题**:
- 迁移是否阻塞读写路径？
- 迁移带宽的 QoS 保障？
- Promotion (冷→热) 的策略是否过于保守？（宁愿多读一次也不愿意误升）
- GPU Direct Storage 的集成可行性？

**Domain Knowledge 映射**:
- `performance/storage-filesystem/KNOWLEDGE.md` — 分级存储、LFS、CXL 跨 SSD 共享
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — CXL、tiered memory、迁移策略
- `performance/gpu-ai-performance/KNOWLEDGE.md` — GDS、GPU 数据搬运

### 4. 碎片整理与空间回收

- **代码入口**: segment 分配与回收逻辑

**分布式系统视角**：
- 固定大小的 segment + 可变大小的对象 = 内部碎片
- 频繁分配释放 + 短生命周期 = 外部碎片
- 类比 LFS (Log-Structured File System)：Mooncake 的写入模式（append-only, immutable）天然适合 log-structured 方式

**碎片化模型**：
```
Segment 大小: 256MB
KV cache 对象分布: 1MB ~ 100MB (取决于序列长度和模型)
小对象 (1-10MB) → 大内部碎片
大对象 (50-100MB) → 跨 segment, 增加管理复杂度
```

**关键问题**:
- 碎片化程度如何量化？是否有碎片率指标？
- Compaction（段合并）的策略和时机？
- 是否可以借鉴 LFS 的 segment cleaning（选择最「空」的 segment 迁移后回收）？
- Segment 回收与驱逐如何协调？（驱逐的对象所在的 segment 何时归还？）

**Domain Knowledge 映射**:
- `performance/storage-filesystem/KNOWLEDGE.md` — LFS, compaction
- `performance/system-tuning/KNOWLEDGE.md` — 内存碎片化

### 5. 存储后端可观测性

- **关键指标**:
  - 各 tier 的容量使用率、命中率（分 layer 统计更有价值）
  - G1→G2、G2→G3 迁移频率、延迟、数据量
  - 分配/释放延迟 (p50/p99)
  - 碎片率（内部碎片、外部碎片分别统计）
  - SSD 的 IOPS、带宽利用率、写入放大因子 (WAF)
  - 跨 NUMA 内存访问比例
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
