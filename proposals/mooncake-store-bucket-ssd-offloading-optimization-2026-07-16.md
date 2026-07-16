# 优化方案: Mooncake Store Bucket Backend SSD Offloading 多维度优化

- **日期**: 2026-07-16
- **目标项目**: Mooncake
- **目标组件**: Store → Storage Backend (Bucket)
- **优化维度**: SSD I/O 路径、驱逐策略、写入路径、元数据管理、KV cache 语义感知
- **引用的领域知识**: SolidAttention(FAST'26), FORGE(OSDI'26), Latte(FAST'26), TapeOBS(FAST'26), Strata(OSDI'26), architecture.md
- **可应用性评级**: 可直接应用 (多项) / 需适配 (部分)
- **基于 Mooncake commit**: 7ee436c (含本地未提交 diff: batch_read + file handle cache)

---

## 问题陈述

Mooncake Store 的 Bucket Backend 是默认的 SSD offloading 存储引擎，负责将 KV cache 从 DRAM (G2) 卸载到 NVMe SSD (G3)。当前实现虽然整体架构合理（bucket 聚合写入、io_uring 读路径、FIFO/LRU 驱逐），但在以下方面存在可测量的优化空间：

1. **写路径页面缓存污染**: 写入不经过 O_DIRECT，数据先进入 page cache → 浪费本可用于 G2 KV cache 的 DRAM
2. **驱逐粒度过粗**: 按整个 bucket (默认 256MB/500 keys) 驱逐 → 冷 bucket 中的热 key 被误杀
3. **元数据 I/O 路径未优化**: 元数据读写不使用 io_uring，初始化扫描串行
4. **缺乏 KV cache 语义感知**: 所有对象同等对待，不区分前缀层（高价值）和尾部层（低价值）
5. **无 one-hit-wonder 过滤**: 写后从未被读的对象仍占满 SSD 空间，直到整桶被驱逐

## 当前实现分析

### Bucket Backend 架构概览

```
写入路径:
  BatchOffload → BuildBucket (serialize to iovecs)
    → PrepareEviction (select+remove oldest buckets via FIFO/LRU)
    → FinalizeEviction (delete evicted bucket files)
    → WriteBucket (io_uring write_aligned with bounce buffer, or vector_write)
    → StoreBucketMetadata (POSIX write, blocking)
    → commit to object_bucket_map_ + buckets_ + lru_index_

读取路径:
  BatchLoad → lookup in object_bucket_map_ (RWLock, shared)
    → get bucket_id + offset + size
    → GetOrOpenFile (file handle cache for reads)
    → [io_uring] batch_read (all keys in same bucket → single io_uring submission)
    → [posix] per-key vector_read (fallback)
    → BucketReadGuard RAII (inflight_reads_ for safe concurrent delete)
```

### 关键代码路径

| 操作 | 文件 | 函数 | 行号 |
|------|------|------|------|
| Bucket 写入 | `storage_backend.cpp` | `BucketStorageBackend::WriteBucket()` | 2020-2158 |
| Bucket 读取 | `storage_backend.cpp` | `BucketStorageBackend::BatchLoad()` | 1396-1578 |
| 驱逐准备 | `storage_backend.cpp` | `BucketStorageBackend::PrepareEviction()` | 2238-2341 |
| 驱逐候选选择 | `storage_backend.cpp` | `BucketStorageBackend::SelectEvictionCandidate()` | 2191-2236 |
| 文件打开 | `storage_backend.cpp` | `BucketStorageBackend::OpenFile()` | 2598-2632 |
| 元数据写入 | `storage_backend.cpp` | `BucketStorageBackend::StoreBucketMetadata()` | 2509-2539 |
| 元数据读取 | `storage_backend.cpp` | `BucketStorageBackend::LoadBucketMetadata()` | 2541-2580 |
| 初始化扫描 | `storage_backend.cpp` | `BucketStorageBackend::Init()` | 1593-1783 |
| 键分桶 | `storage_backend.cpp` | `BucketStorageBackend::GroupOffloadingKeysByBucket()` | 1895-1986 |
| 文件创建 | `storage_backend.cpp` | `StorageBackend::create_file()` | 753-790 |
| 配置解析 | `storage_backend.cpp` | `BucketBackendConfig::FromEnvironment()` | 60-86 |

### 关键常量与默认值

| 配置项 | 默认值 | 环境变量 |
|--------|--------|---------|
| bucket_size_limit | 256 MB | `MOONCAKE_OFFLOAD_BUCKET_SIZE_LIMIT_BYTES` |
| bucket_keys_limit | 500 | `MOONCAKE_OFFLOAD_BUCKET_KEYS_LIMIT` |
| eviction_policy | `fifo` (最近改为默认) | `MOONCAKE_OFFLOAD_BUCKET_EVICTION_POLICY` |
| max_total_size | 0 (自动: 90% 磁盘容量) | `MOONCAKE_OFFLOAD_BUCKET_MAX_TOTAL_SIZE` |
| use_uring | false | `MOONCAKE_OFFLOAD_USE_URING` |
| O_DIRECT for writes | **禁用** (仅读路径使用) | (硬编码) |
| File handle cache | **本地 diff 中** (仅读模式) | (硬编码) |
| io_uring batch_read | **本地 diff 中** | (编译时 USE_URING) |

### 本地未提交 diff 分析

当前工作区有一个未提交的 `storage_backend.cpp` 修改，包含两项优化：
1. **File handle cache (`GetOrOpenFile`)**: 缓存读模式的 bucket 文件句柄，避免每次 `BatchLoad` 重复 `open()` 系统调用
2. **Batch read (`batch_read`)**: 将同一 bucket 内的多个 key 读取合并为单次 io_uring 提交 → 利用 NVMe 多队列并行

这两项都是正确方向的优化。本方案在此基础上进一步提出改进。

---

## 可应用的论文洞察

### 洞察 1: Page Cache 对 KV Cache SSD Offloading 是净负收益（来源：Latte(FAST'26) + architecture.md §2.2）

- **论文中的做法**: Latte 使用 O_DIRECT 完全绕过 page cache，因为 cloud storage 场景下数据没有复用价值——第一次读后不会被再次访问。Latte 还通过 SPDK 用户态 NVMe 驱动消除内核上下文切换。
- **当前实现的差距**: 
  - `WriteBucket()` 的写入路径不启用 O_DIRECT (`OpenFile()` 仅在 `mode == FileMode::Read` 时加 `O_DIRECT`)
  - 代码注释明确说 "write latency is not sensitive in this scenario, and O_DIRECT writes require 4096-byte alignment padding which corrupts meta file parsing and wastes disk space on data files" 
  - 但这个理由不完全成立：(1) data file 末尾的 padding 浪费可以忽略 (最多 4095 bytes/bucket)，(2) 真正的问题是 page cache 污染——写入的数据占据了 DRAM，而 KV cache 的 offload 数据在被读回 G2 之前基本没有 page cache 复用价值
  - io_uring 路径虽然用了 `write_aligned` + bounce buffer，但数据仍在写入前被拷贝到对齐缓冲区（额外的 memcpy 开销），且写入后数据仍留在 page cache
- **可迁移性分析**: **可直接应用**。方案：(a) 为 bucket data file 写入启用 O_DIRECT，(b) 或者使用 `posix_fadvise(POSIX_FADV_DONTNEED)` 在写入后立即释放 page cache，(c) 利用 io_uring 的 `IOSQE_FIXED_FILE` 减少 fd 操作开销。元数据文件（`.meta`）因为小且频繁重读（Init 时扫描），保留 buffered I/O 是合理的。

### 洞察 2: 按生命周期分组写入 → 几乎零 GC 开销（来源：TapeOBS(FAST'26)）

- **论文中的做法**: TapeOBS 按 3 月粒度的生存期将对象分组到同一磁带。因为同组对象同时被删除 → GC 时整盘磁带可直接回收，无需读有效数据+重写。异步缓冲池使分组称为可能。
- **当前实现的差距**: 
  - Mooncake 的 bucket 驱逐是整桶删除（这个决策是对的！），但 bucket 内的对象生命周期是**随机的**——不同 TTL 的对象混在同一个 bucket 中
  - 结果：一个 bucket 中只要有一个 key 被频繁访问（延长了生命周期），整桶都无法被驱逐（对于 LRU 策略）
  - 对于 FIFO 策略，一个 bucket 中的所有 key 被同时驱逐，无论它们的实际价值
  - 当前 `GroupOffloadingKeysByBucket` 仅按 size+count 限制做贪心填桶，完全不考虑生命周期亲和性
- **可迁移性分析**: **需适配**。KV cache 的生命周期比磁带归档短得多（秒到分钟），但原理相同。改进方向：
  - 在 `GroupOffloadingKeysByBucket` 中增加 TTL 亲和性：相近 TTL 的对象放入同一 bucket
  - 或者在 heartbeat 驱动的 offload 批次中自然按时间窗口分批（当前已部分做到——每个 heartbeat 周期的 offload 对象一起写入同一 bucket）
  - 当前实现实际上已经有隐式的"时间窗口分组"：每次 `BatchOffload` 调用创建一个新 bucket。只要调用频率合理，同一 bucket 内的对象确实有相近的创建时间。但 TTL 可能差异大。

### 洞察 3: S3-FIFO Candidate Queue 过滤 one-hit-wonder（来源：Latte(FAST'26)）

- **论文中的做法**: 72% 的 I/O trace 对象只被访问一次。S3-FIFO 用一个小型 ghost/candidate queue：首次访问只记录元数据（不占主缓存空间），第二次访问才 promote 到主缓存 → 缓存命中率 >82%。Latte 用 Linear SVM 做 per-I/O 路由决策（200ns 推理）。
- **当前实现的差距**:
  - Bucket backend 将所有 offload 对象一视同仁地持久化到 SSD
  - 如果某个 KV cache 对象被 offload 到 SSD 后从未被读回（one-hit-wonder），它仍然占据 bucket 空间直到整桶被驱逐
  - 没有机制在写入前判断"这个对象值得 offload 到 SSD 吗？"
  - 不过，offload 决策是在 Master 端做的（heartbeat 返回 offloading_objects），Backend 端只负责执行。所以这个过滤应该在 Master 端或 FileStorage 层做
- **可迁移性分析**: **需适配**。改进方向：
  - 在 `FileStorage::OffloadObjects()` 或 Master 的 offload 决策中，引入一个轻量的 "offload-worthiness" 判断：只 offload 那些被访问超过 N 次或有 prefix match 潜力的对象
  - 对于 bucket backend，可以增加一个 **delayed offload** 机制：对象先进入一个 DRAM 中的观察窗口，在窗口内被再次访问则保留在 DRAM，窗口过期后才真正 offload 到 SSD
  - 这个改动涉及 Master ↔ Client 的交互协议，不纯粹是 storage-backend 层面的优化

### 洞察 4: 分层价值感知驱逐（来源：architecture.md §4.1 + SolidAttention(FAST'26)）

- **论文中的做法**: SolidAttention 证明 LLM 推理中 attention sparsity 高度集中在少数 token——81% 跨迭代选择相似度。这意味着 prefix 层的 KV cache 被命中的概率远高于尾部层。分层驱逐（优先驱逐尾部层、保留前缀层）可比 uniform 驱逐显著提高缓存命中率。
- **当前实现的差距**:
  - Bucket backend 的驱逐策略完全**不感知 KV cache 的层/位置语义**
  - FIFO 按 bucket 创建时间驱逐，LRU 按 bucket 最后访问时间驱逐
  - 没有机制区分 bucket 内 prefix 层 vs 尾部层的 KV cache
  - 也没有在 bucket 间按"token 位置价值"加权排序
- **可迁移性分析**: **需适配**。两种可能的实现深度：
  - **浅层方案 (P2)**: 在 `StorageObjectMetadata` 中增加 `token_position` 或 `layer_range` 字段，驱逐时给 prefix 层对象更高的保留权重。但 bucket 粒度驱逐使 per-object 权重难以直接应用——需要 bucket compaction
  - **深层方案 (未来)**: 实现 per-object eviction（而非 per-bucket），但这需要根本性重构 bucket 存储模型

### 洞察 5: io_uring 写路径优化与元数据异步化

- **来源**: 代码分析 + Strata(OSDI'26) GPU-assisted I/O 思想
- **当前实现的差距**:
  - 数据写入已部分使用 io_uring（`write_aligned`），但需要 bounce buffer memcpy
  - 元数据写入 (`StoreBucketMetadata`) 使用 POSIX `write()`，是同步阻塞调用
  - 元数据读取 (`LoadBucketMetadata` → `OpenFile(mode=Read)` → `PosixFile`) 不使用 io_uring，即使 IO_URING 编译开启
  - `Init()` 中的元数据扫描是串行的——每个 `.meta` 文件逐个读取和解析
- **可迁移性分析**: **可直接应用**。改进方向：
  - 元数据文件也走 io_uring 异步读写
  - Init() 阶段使用 io_uring batch 读取多个 `.meta` 文件
  - 利用 `IOSQE_IO_LINK` 链接 data write + meta write + fsync，减少 submission 次数

### 洞察 6: 写入优化——消除 bounce buffer memcpy

- **来源**: 代码分析
- **当前实现的差距**: 在 `WriteBucket()` 的 io_uring 路径中（行 2073-2078），所有 iovecs 的数据被 memcpy 到一个对齐的 bounce buffer，然后再 `write_aligned`。这有两个问题：
  1. **额外的 memcpy 开销**: KV cache 对象通常较大（MB 级），memcpy 不可忽略
  2. **bounce buffer 大小限制**: 默认 32MB，超过后需要临时分配并 log warning
- **可迁移性分析**: **可直接应用**。改进方向：
  - 使用 `io_uring_prep_writev()` 而非 `write_aligned`，直接传递 iovecs——让内核处理 gather 写入
  - 或者在上层（`BuildBucket`）就直接将数据写入一个对齐的连续缓冲区，避免二次拷贝
  - 如果必须用 O_DIRECT + 非对齐写入，可以用 `RWF_UNCACHED` + `pwritev2` 作为备选

---

## 建议修改

### 1. [P0] 写路径启用 Direct I/O，消除 page cache 污染

**改动范围**: `storage_backend.cpp:OpenFile()`, `storage_backend.cpp:WriteBucket()`

```cpp
// OpenFile() 中: 对 Write 模式也启用 O_DIRECT (当 use_uring 开启时)
#ifdef USE_URING
    if (file_storage_config_.use_uring) {
        // Write also uses O_DIRECT to avoid page cache pollution.
        // KV cache data has no re-read locality from page cache,
        // and the DRAM is better used for G2 (hot KV cache).
        flags |= O_DIRECT;
        bool use_direct_io = true;
        return std::make_unique<UringFile>(path, fd, 32, use_direct_io);
    }
#endif
```

- **风险缓解**: 对于 `.meta` 文件，保持 buffered I/O（元数据小且需要频繁重读）
- **预期收益**: 释放 page cache 占用的 DRAM，增加 G2 KV cache 可用容量（取决于写入吞吐量，可能回收数百 MB~GB 级别的 page cache）
- **注意**: 本地 diff 中的 `batch_read` 已使用 O_DIRECT 读路径，配合此项实现读写全路径 Direct I/O

### 2. [P1] 元数据 I/O 使用 io_uring + Init 批量加载

**改动范围**: `storage_backend.cpp:StoreBucketMetadata()`, `LoadBucketMetadata()`, `Init()`

- `StoreBucketMetadata()`: 元数据写入也走 io_uring `write_aligned`（metadata 已通过 protobuf 序列化到连续 string，天然对齐友好）
- `LoadBucketMetadata()`: 使用 io_uring 读取
- `Init()`: 批量收集所有 `.meta` 文件路径 → 使用 io_uring batch 读取+解析，而非逐个文件的串行 read
- **预期收益**: Init 启动延迟降低 30-50%（取决于 bucket 数量），元数据写入延迟降低

### 3. [P1] 细粒度驱逐: Bucket 热度评分 + 冷热分离

**改动范围**: `storage_backend.cpp:PrepareEviction()`, `SelectEvictionCandidate()`, 新增 `BucketHeatScore()`

- 为每个 bucket 维护访问计数（不仅是 boolean 的 last_access，而是 `read_count`）
- `SelectEvictionCandidate()` 在选择驱逐候选时考虑：
  - FIFO 模式下：加入最小访问计数阈值——即使 bucket 最老，如果近期有大量读取也跳过
  - LRU 模式下：已有的 lazy index repair 思路正确，增加 bucket 内 active key 比例作为 tiebreaker
- 可选增强：在驱逐前扫描 bucket 中的 hot keys → 将它们提升到新 bucket（compaction-like）
- **预期收益**: 减少热 KV cache 被误驱逐的概率，提升 SSD→DRAM promotion 命中率

### 4. [P2] Bucket 打包优化: TTL 亲和 + 更优 bin-packing

**改动范围**: `storage_backend.cpp:GroupOffloadingKeysByBucket()`

- TTL 亲和：将相近 TTL 的对象尽量放入同一 bucket（当前隐式做到了时间窗口分组，可显式化）
- Bin-packing 改进：当前是贪心 first-fit，可用 first-fit-decreasing（按 size 降序排序后再填）提高 bucket 空间利用率
- 增加 `ungrouped_offloading_objects_` 的跨批次合并逻辑，减少碎片化的小 bucket
- **预期收益**: 减少 bucket 内部碎片，提高 90% 磁盘配额的实际利用率

### 5. [P2] One-hit-wonder 过滤: Delayed Offload 观察窗口

**改动范围**: `file_storage.cpp:OffloadObjects()`, 或在 Master 端 offload 决策

- 在 SSD offloading 前增加一个 DRAM 中的小型观察窗口（如 512 个 key 的 LRU）
- 对象第一次被标记为 offload 候选时，不立即写入 SSD → 先放入观察窗口
- 在观察窗口内被再次访问 → 取消 offload（保留在 DRAM）
- 窗口过期或满 → 真正 offload 到 SSD
- **预期收益**: 减少 SSD 写入量（估计 30-50% 的对象可能不需要持久化），同时减少 SSD 读回延迟

### 6. [P2] 写入路径消除 bounce buffer memcpy

**改动范围**: `storage_backend.cpp:WriteBucket()` (io_uring 分支)

- 方案 A: 使用 `io_uring_prep_writev()` 替代 `write_aligned()`，配合 `RWF_UNCACHED` 绕过 page cache
- 方案 B: 将 `BuildBucket()` 的输出从 `std::vector<iovec>` 改为直接写入对齐的连续缓冲区
- 方案 A 的侵入性更小，优先尝试
- **预期收益**: 减少每个 bucket 写入的 memcpy 开销（bucket 256MB → 省 256MB 的 memcpy）

---

## 预期收益

| 优化项 | 延迟 | 吞吐 | DRAM 可用性 | SSD 寿命 | 置信度 |
|--------|------|------|------------|---------|--------|
| P0: Direct I/O 写入 | — | — | +数百 MB~GB (page cache 回收) | ↓写放大 (减少 double-buffering) | 高 |
| P1: 元数据 io_uring | 写入 P99 -20~30% | — | — | — | 高 |
| P1: 细粒度驱逐 | — | — | — | — (提升命中率) | 中 |
| P2: Bucket 打包 | — | — | +5~10% 有效容量 | — | 中 |
| P2: One-hit-wonder 过滤 | promotion 延迟 - (避免不必要读回) | SSD 写量 -30~50% | — | +SSD 寿命 | 中 |
| P2: 消除 memcpy | 写入延迟 -10~20% | — | — | — | 中 |

- **综合 DRAM 回收**: P0 的 Direct I/O 写入 + P2 one-hit-wonder 过滤可能回收 **数百 MB ~ 1GB+** 的 DRAM，可用于 G2 热 KV cache → 直接提升 cache 命中率
- **综合延迟**: 元数据 I/O 异步化 + 消除 memcpy 使单次 offload 延迟降低 20-40%

---

## 风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| O_DIRECT 写入需要所有 iovec 对齐到 4096 | 中 | 写入失败，bucket 损坏 | 在 `BuildBucket()` 阶段就做对齐检查，不对齐的对象走 bounce buffer fallback；已有的 `write_aligned` 路径已处理对齐 |
| O_DIRECT 写导致小 bucket 的空间放大 | 低 | 磁盘利用率下降 <1% | bucket 默认 256MB，padding 最多 4095 bytes → 放大 <0.002% |
| 元数据 io_uring 异步化引入并发 bug | 低 | 元数据不一致 | metadata 文件小（KB 级），可保持 io_uring 同步完成（submit + wait 在同一调用栈） |
| 细粒度驱逐增加 CPU 开销 | 低 | heartbeat 路径延迟增加 | 仅在驱逐触发时计算（低频操作），访问计数用 atomic 无锁递增 |
| TTL 亲和分桶可能延迟 offload（等同类 TTL 凑满一桶） | 中 | 对象在 DRAM 中停留更久 | 设置最大等待时间，超时后降级为普通填桶 |

---

## 验证方案

### 基准测试

1. **写入路径 Direct I/O**: 
   - 对比 `MOONCAKE_OFFLOAD_USE_URING=1` 下的 page cache 占用（`/proc/meminfo` 的 `Cached` 字段）
   - 监控指标: `ssd_write_latency_us` (已有) + 新增 `page_cache_kb_after_write`
   
2. **元数据 io_uring**:
   - 测量 `Init()` 扫描 1000/10000 bucket 的耗时
   - `StoreBucketMetadata` 的 p50/p99 延迟

3. **驱逐命中率**:
   - 跟踪 `ssd_total_ops` / `eviction_count` 比值
   - 新增指标: `promotion_hit_rate`（从 SSD 读回后又在 DRAM 中被命中的比例）

4. **One-hit-wonder 过滤**:
   - 统计 offload 后 N 秒内被读回的比例
   - 如果比例 <30%，说明过滤有效

### A/B 测试配置

```bash
# P0: Direct I/O for writes
# 无需额外配置，代码改动后自动生效（需 USE_URING=1）
export MOONCAKE_OFFLOAD_USE_URING=true

# P1-P2: 新配置项
export MOONCAKE_OFFLOAD_BUCKET_EVICTION_POLICY=lru  # 已有
export MOONCAKE_OFFLOAD_BUCKET_TTL_AFFINITY=true     # 新增: TTL 亲和分桶
export MOONCAKE_OFFLOAD_DELAYED_OFFLOAD_WINDOW=128   # 新增: 观察窗口大小
```

---

## 附录：与已有本地 diff 的关系

当前工作区的本地 diff (file handle cache + batch_read) 是本方案中读路径优化的基础。建议：

1. 先 commit 本地 diff 作为独立的读路径优化
2. 本方案中的 P0（Direct I/O 写）和 P1（元数据 io_uring）在其基础上构建
3. P2 优化可以与 P0/P1 并行开发

## 参考文献

- SKILL.md: `mooncake/store/storage-backend/SKILL.md` (优化维度 2/3/5)
- KNOWLEDGE.md: `mooncake/store/storage-backend/KNOWLEDGE.md` (初始优化目标)
- architecture.md: §2.2 存储层次, §4.1 KV cache 语义, §4.4 端到端延迟模型
- Domain knowledge: `performance/storage-filesystem/KNOWLEDGE.md` (Latte, TapeOBS, Helmsman)
- Domain knowledge: `performance/gpu-ai-performance/KNOWLEDGE.md` (SolidAttention, Strata, ECHO)
- Domain knowledge: `architecture/memory-storage-hierarchy/KNOWLEDGE.md` (Duhu, Blowfish)
- 阅读笔记: `common/knowledge-synthesis/notes/SolidAttention(FAST'26).md`
