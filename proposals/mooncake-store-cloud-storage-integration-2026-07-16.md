# 优化方案: Mooncake Store 对接云对象存储 (S3/OSS/COS) 实现方案

- **日期**: 2026-07-16
- **目标项目**: Mooncake
- **目标组件**: Store → storage-backend (含 distributed 子目录) + client
- **优化维度**: 存储后端扩展 (云存储适配)
- **引用的领域知识**: Latte(FAST'26), TapeOBS(FAST'26), ACOS(FAST'26), OBASE(OSDI'26), DistributedStorageBackend (现有模式)
- **可应用性评级**: 需适配 (核心架构模式已存在,需补充云对象存储适配器和调度策略)
- **基于 Mooncake commit**: 7ee436c

---

## 问题陈述

Mooncake Store 当前支持四种存储后端: `kBucket` (本地 SSD)、`kFilePerKey` (本地文件)、`kOffsetAllocator` (sharded 分配器)、`kDistributed` (分布式文件系统,仅 hf3fs/3FS 适配器)。但**缺少对云对象存储 (AWS S3、阿里云 OSS、腾讯 COS、GCS、Azure Blob) 的原生支持**。

云对象存储对以下场景有明确价值:
1. **弹性容量**: KV cache offload 可弹性扩展远超本地 SSD 的容量 (TB → PB)
2. **跨节点共享**: 一个节点的 offload 数据可被其他节点直接读取 (无需跨节点 SSD RPC)
3. **降低本地存储压力**: 冷 KV cache 可迁移到云存储，本地 SSD 仅保留热数据 → 类似 Latte 的"本地盘做缓存 + 云盘做持久层"
4. **成本优化**: 云对象存储 $/GB 远低于本地 NVMe SSD (类似 ACOS 的 RF 优化逻辑)

当前 Mooncake 已有很好的架构基础——`DistributedStorageBackend` + `FileSystemAdapter` 的抽象模式可以直接复用，只需新增云对象存储适配器。

## 当前实现分析

### 1. 存储后端抽象 (storage_backend.h:296-346)

```cpp
class StorageBackendInterface {
    virtual tl::expected<void, ErrorCode> Init() = 0;
    virtual tl::expected<int64_t, ErrorCode> BatchOffload(...) = 0;
    virtual tl::expected<void, ErrorCode> BatchLoad(...) = 0;
    virtual tl::expected<bool, ErrorCode> IsExist(const std::string& key) = 0;
    virtual tl::expected<bool, ErrorCode> IsEnableOffloading() = 0;
    virtual tl::expected<void, ErrorCode> ScanMeta(...) = 0;
    virtual tl::expected<std::vector<std::string>, ErrorCode>
        EvictAboveDiskWatermark(...) { return std::vector<std::string>{}; }
};
```

### 2. DistributedStorageBackend 模式 (distributed_storage_backend.h)

这是**关键的参考实现**——已经展示了如何通过 `FileSystemAdapter` 接口将存储后端与具体文件系统解耦:

```cpp
class DistributedStorageBackend : public StorageBackendInterface {
    std::unique_ptr<FileSystemAdapter> fs_adapter_;
    // BatchOffload → fs_adapter_->VectorWriteFile()
    // BatchLoad → fs_adapter_->ReadFile()
    // IsExist → fs_adapter_->FileExists()
    // ScanMeta → fs_adapter_->ListFilesWithInfo()
};
```

### 3. FileSystemAdapter 接口 (fs_adapter.h:26-110)

```cpp
class FileSystemAdapter {
    virtual tl::expected<size_t, ErrorCode> WriteFile(path, data) = 0;
    virtual tl::expected<size_t, ErrorCode> ReadFile(path, buf, len) = 0;
    virtual tl::expected<size_t, ErrorCode> VectorWriteFile(...) = 0;
    virtual tl::expected<size_t, ErrorCode> VectorReadFile(...) = 0;
    virtual tl::expected<void, ErrorCode> DeleteFile(path) = 0;
    virtual tl::expected<bool, ErrorCode> FileExists(path) = 0;
    virtual tl::expected<vector<string>, ErrorCode> ListFiles(dir) = 0;
    virtual tl::expected<vector<FileInfo>, ErrorCode> ListFilesWithInfo(dir);
};
```

### 4. 现有 S3 使用: Snapshot Object Store (仅用于 HA 备份,不用于数据路径)

```
mooncake-store/src/ha/snapshot/object/snapshot_object_store.cpp  ← 快照备份
mooncake-store/src/utils/s3_helper.cpp                            ← AWS SDK 封装
```

`S3Helper` 已实现: `UploadBuffer()`, `DownloadBuffer()`, `ListObjectsWithPrefix()`, `DeleteObjectsWithPrefix()`, multipart 上传/下载。但这是为快照场景设计的 (低频、大文件、完整性优先)，**不适用于 KV cache 数据路径** (高频、小对象、延迟敏感)。

### 5. 后端工厂 (storage_backend.cpp:4288-4345)

```cpp
switch (config.storage_backend_type) {
    case StorageBackendType::kBucket:        → BucketStorageBackend
    case StorageBackendType::kFilePerKey:    → StorageBackendAdaptor
    case StorageBackendType::kOffsetAllocator: → OffsetAllocatorStorageBackend
    case StorageBackendType::kDistributed:   → DistributedStorageBackend + FileSystemAdapter
}
```

需要在 `StorageBackendType` 枚举中新增 `kCloudObject` 或在 `kDistributed` 下新增 cloud adapter。

## 可应用的论文洞察

### 洞察 1: Latte 的本地-云混合架构模式 (Latte, FAST'26)

- **论文中的做法**: 用本地 NVMe SSD 做高性能缓存 (S3-FIFO 管理) + 标准 EBS 做可靠持久后端。写入时 ML dispatcher 二分类路由 (本地 cache vs 远程 backend)，本地仅保留 >82% 命中率的热数据
- **当前实现的差距**: Mooncake 的 offload 路径只能写到本地 SSD 或 3FS 分布式文件系统，没有对云对象存储的后备层。`FileStorage` 直接管理 `StorageBackendInterface` — 仅支持单一后端
- **可迁移性分析**:
  - **核心模式可直接复用**: Mooncake 的 G2(DRAM)↔G3(SSD) 分级 → 扩展为 G2(DRAM)↔G3(local SSD)↔G4(cloud object store)
  - **差异**: Latte 关注块存储(virtual disk)，Mooncake 关注对象存储(KV cache blob)。对象语义更匹配云对象存储的原生接口
  - **增量迁移**: 不需要改造现有 G3 本地 SSD 路径，只需在 SSD 水位到达阈值时触发 offload 到云存储

### 洞察 2: TapeOBS 的全异步 + HDD 缓冲机制 (TapeOBS, FAST'26)

- **论文中的做法**: HDD 池做持久写缓冲 (吸收突发写入, 提供故障容错), DataBrain 按生存期分批 flush 到磁带——"全异步 = 可调度"的架构模式
- **当前实现的差距**: Mooncake 的 `BatchOffload()` 是同步调用的——本地 SSD offload 延迟可忽略但云存储延迟 10-100ms+
- **可迁移性分析**:
  - **不一定要全异步**: 由于 Mooncake 的 offload 操作在独立线程中执行 + KV cache 写路径不走临界区 (decouple put from offload)，同步写云存储不会阻塞推理
  - **核心启发**: 引入 `CloudWriteBuffer`——本地 SSD 先吸收 writes，后台按 bucket 或 TTL 分组批量上传到云端——利用云存储的批量降低 API 调用成本 (S3 PUT 按请求数计费)

### 洞察 3: OBASE 的对象级冷热分离 (OBASE, OSDI'26)

- **论文中的做法**: 对象级热度追踪 (而非 page 级), 冷热对象聚类到不同堆 → page-based tiering 效率提升 2-4×
- **当前实现的差距**: Mooncake 的驱逐/offload 是 per-bucket (256MB, ≤500 keys) 或 per-object 级别，但 offload 决策是 FIFO/LRU 全局的，不感知单个 key 的访问频率
- **可迁移性分析**:
  - **对象级热度指标**: 在 offload 到云存储之前，先判断对象是否"值得保留在本地"——只把真正的冷数据推到云端
  - 可结合现有 `CountMinSketch` (`count_min_sketch.h`) 做 per-key 访问频率估计
  - **S3-FIFO (Latte) 与 OBASE 结合**: candidate queue 过滤 one-hit-wonder → 避免云存储被一次性 flush 的对象污染

### 洞察 4: ACOS 的 immutable sealed container (ACOS, FAST'26)

- **论文中的做法**: 一旦 container sealed → 不可变 → 跨域迁移变为纯文件复制 (零对象级重建)
- **当前实现的差距**: Mooncake 的 KV cache 对象本身就是 immutable-after-put——这个特性尚未被用于简化云存储集成
- **可迁移性分析**:
  - **Immutable 对象 → 无一致性问题**: S3 overwrite 的 replication delay 不会影响 Mooncake (永远不会 overwrite 同一个 key)
  - **Sealed container = Bucket**: Mooncake 的 Bucket 已经是天然的 "sealed container"——写满后不再修改 → 可作为云存储上传的基本粒度
  - **批量删除**: Bucket 内的所有 key 大概率同时过期 → 云存储上用 delete-objects (批量删除, 免费) 替代 per-key delete

### 洞察 5: S3 Model-Based Testing 的回归防护 (S3 MBT, OSDI'26)

- 启发: 云存储适配器的 core API 正确性 (PUT/GET/LIST/DELETE) 可以通过 model-based/differential testing 来验证
- 在 Mooncake 中体现为: 新 cloud adapter 应通过与相同 data set 下本地 KV backend 的 comparison test (differential testing)

## 建议修改

### P0 — 新增 CloudObjectStoreAdapter (实现 FileSystemAdapter 接口)

这是最小可行方案——最大程度复用现有 `DistributedStorageBackend` 模式。

```
mooncake-store/
├── include/storage/distributed/
│   ├── fs_adapter.h                              # 不变
│   ├── cloud_object_store_adapter.h              # [NEW]
│   └── distributed_storage_backend.h             # 微调: 新增 cloud_ 配置字段
├── src/storage/distributed/
│   ├── distributed_storage_backend.cpp            # 微调: 适配新 adapter
│   └── cloud_object_store_adapter.cpp            # [NEW]
└── src/storage_backend.cpp                        # 微调: 工厂注册 cloud backend
```

`CloudObjectStoreAdapter` 核心接口映射:

| FileSystemAdapter | → S3 API |
|---|---|
| `WriteFile(path, data)` | `PutObject(key, data)` |
| `ReadFile(path, buf, len)` | `GetObject(key)` → `buf` |
| `VectorWriteFile(path, iov, iovcnt, offset)` | `PutObject(key, combined_iov)` |
| `VectorReadFile(path, iov, iovcnt, offset)` | `GetObject(key)` with range GET |
| `DeleteFile(path)` | `DeleteObject(key)` |
| `FileExists(path)` | `HeadObject(key)` → 200/404 |
| `ListFiles(dir)` | `ListObjectsV2(prefix=dir/)` with pagination |
| `GetFileSize(path)` | `HeadObject(key)` → Content-Length |
| `Init(mount_path)` | Init SDK, resolve credentials, validate bucket access |
| `Shutdown()` | SDK shutdown |
| `GetName()` | 返回 "s3" / "oss" / "cos" / "gcs" |

**关键实现细节**:

1. **认证链**: 标准 SDK credential chain (环境变量 → instance profile/STS → 配置文件)
2. **Multipart Upload**: 阈值 5MB (S3 minimum part size), 大 KV cache 对象自动分片上传
3. **连接池**: 复用 `Aws::S3::S3Client` (内部 HTTP 连接池), 避免 per-request 建连
4. **重试策略**: 指数退避 + jitter (SDK 内置), 对 throttling (503 SlowDown) 特殊处理
5. **Key 编码**: 复用 `EscapeFilename()` → key 中不允许的字符转义; 加前缀组织 (如 `kv_cache/<hash_prefix>/<escaped_key>`)

**配置** (`CloudObjectStoreConfig`):

```cpp
struct CloudObjectStoreConfig {
    std::string endpoint;            // S3 endpoint (可为空，用 SDK 默认)
    std::string bucket;              // bucket 名
    std::string region = "us-east-1";
    std::string access_key;          // 或走 IAM role / STS
    std::string secret_key;
    std::string key_prefix = "mooncake/kv_cache/";  // 对象 key 前缀
    int64_t multipart_threshold = 5 * 1024 * 1024;  // 5MB
    int max_retries = 3;
    int connect_timeout_ms = 3000;
    int request_timeout_ms = 30000;
    bool use_virtual_hosted_style = true;
    bool enable_ssl = true;
};
```

### P1 — 扩展为混合后端 (Latte 模式)

当前 `RealClient` 通过 `FileStorage` 使用单一 `StorageBackendInterface`。扩展为 **两级后端**:

```
G3: 本地 SSD (现有 BucketStorageBackend 等)
  → 高水位触发 offload →
G4: 云对象存储 (CloudObjectStoreAdapter)
```

**触发策略**:
1. **容量水位**: 本地 SSD 达到 `disk_eviction_high_watermark_ratio` → 将最冷的 buckets/keys 迁移到云存储 (而非仅驱逐)
2. **TTL 分层**: 长 TTL (> 5min) 的对象写入云存储, 短 TTL 保留在本地 SSD
3. **Access frequency**: 通过 CountMinSketch + FIFO/LRU bucket 统计 → 低访问频率 bucket 整体迁移

**代码变更**:
- `RealClient` 或 `FileStorage` 新增 `StorageBackendInterface* cloud_backend_` 成员
- Offload 路径增加路由逻辑: `should_offload_to_cloud(key) ? cloud_backend_->BatchOffload() : local_backend_->BatchOffload()`
- Read 路径: 先在 `local_backend_` 查找 → miss 则 fallback 到 `cloud_backend_`

### P2 — 写入缓冲与批量上传 (TapeOBS 模式)

在本地 SSD 上维护一个 **cloud write buffer (bucket 粒度)**:

1. 标记为 offload-to-cloud 的 keys 先写入本地 SSD buffer area
2. 当 buffer bucket 满 (256MB 或 500 keys) → 打包整体上传到云存储
3. 上传完成后, 本地 buffer 中的 keys 标记为 `REMOTE` 并从本地 SSD 回收空间
4. 读路径: 先查本地 index → `REMOTE` → 从云存储下载

**收益**: 利用批量上传减少 API cost (一个 bulk request vs N 个 per-key requests)

### P3 — SDK 多厂商适配 (可选)

Mooncake 目前已编译可选 AWS SDK (通过 `HAVE_AWS_SDK`)。对于非 S3 兼容的对象存储:
- 阿里云 OSS: 与 S3 API 高度兼容 → 可直接用 S3 SDK + 自定义 endpoint
- 腾讯 COS: 同上
- GCS: 需要 `google-cloud-cpp` SDK → 新增 `GcsObjectStoreAdapter`
- Azure Blob: 需要 `azure-storage-blobs-cpp` SDK → 新增 `AzureBlobAdapter`

建议分阶段: 先用 **S3-compatible adapter** 覆盖 90% 场景, 后续视需求增加原生 SDK adapter。

## 预期收益

| 指标 | 预期收益 | 置信度 |
|------|---------|--------|
| **G3 可用容量** | +∞ (理论上限 = 云存储 bucket 配额) | 高 |
| **本地 SSD 利用率** | 保持在高水位以下 (80-90%)，避免 eviction thrashing | 高 |
| **跨节点数据共享** | 消除跨节点 SSD RPC 读取开销 (延迟从 ~1ms 变为 ~10-50ms，但带宽不受限) | 中 |
| **存储成本** | 冷 KV cache 从 NVMe SSD (~$0.1-0.3/GB/月) → S3 Standard (~$0.02/GB/月), ~5-15× 降低 | 高 |
| **写延迟 (P99)** | 如使用同步云写入，P99 ↑10-100ms；如使用本地缓冲异步写，P99 不变 | 取决于实现 |
| **读延迟 (首次)** | 首次从云读取 ~10-100ms vs 本地 SSD ~100µs → 对 cache miss 场景可接受 | 高 |

## 风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| 云存储延迟导致 offload 路径阻塞推理 | 中 | Prefill 延迟 P99 恶化 | 异步 offload 线程 + 本地 SSD 做 write buffer；可变配置的 timeout |
| S3 API 费用过高 (per-request 计费) | 中 | TCO 优势被抵消 | Bucket 粒度批量上传 (合并 N 个 key→1 次 PUT)；S3 Intelligent-Tiering 自动分层 |
| 云存储厂商锁定 | 低 | 难以迁移到不同云 | 优先 S3-compatible adapter (覆盖 OSS/COS/MinIO)；多 adapter 接口统一 |
| 网络分区导致云存储不可写 | 低 | offload 停滞，本地 SSD 满 | 本地 SSD 25-30% headroom 作为缓冲区 (TapeOBS 模式)；fallback 到 eviction |
| 读延迟 P99 恶化 (首次从云读取) | 中 | Decode 延迟受 KV cache read 影响 | 仅对低频 key 走云存储 (类似 S3-FIFO 的 candidate queue)；预测性 prefetch |
| AWS SDK 编译依赖增大二进制 | 低 | 构建复杂度增加 | 通过 CMake feature flag `USE_CLOUD_STORAGE` 控制，默认关闭 |

## 验证方案

### 阶段 1: 单元测试 (CloudObjectStoreAdapter)
- 本地 MinIO 容器 (S3-compatible) 作为测试 target
- 覆盖: `WriteFile` → `ReadFile` → `FileExists` → `DeleteFile` → `ListFiles` 完整生命周期
- Differential test: 相同 data set → CloudObjectStoreAdapter vs 本地 BucketStorageBackend → 结果一致

### 阶段 2: Benchmark
- `store_bench.cpp` 增加 cloud backend 模式
- 指标: offload throughput (keys/sec), read throughput, P50/P99 latency
- 对比: 仅本地 SSD vs 仅云存储 vs 混合 (local hot + cloud cold)

### 阶段 3: 集成测试
- vLLM + Mooncake 完整链路 (SGLang HiCache 同理)
- Workload: ShareGPT / LMSys-Chat-1M trace replay
- 验证: 缓存命中率、端到端延迟 (TTFT/TPOT)、本地 SSD 水位稳定性

### 阶段 4: 监控指标
- `cloud_offload_ops_total` / `cloud_offload_bytes_total`
- `cloud_read_ops_total` / `cloud_read_bytes_total`
- `cloud_api_latency_p50/p99`
- `cloud_api_error_rate` (按 HTTP status code 分类)
- `local_ssd_watermark_ratio` (监控是否保持在 75-85% 健康区间)
