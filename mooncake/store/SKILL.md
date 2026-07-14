# Mooncake Store Optimization

Mooncake Store 是分布式 KV 缓存对象存储层，负责 KVCache 的存储、检索和生命周期管理。采用 Master/Client 架构，支持三级存储（G1 GPU HBM / G2 CPU DRAM / G3 NVMe SSD）。

## 源代码地图

参见 `mooncake/repo-map.md` § Mooncake Store。

### 分析入口点

1. **Master 服务**: `src/mooncake_master/master_service.cpp` — 元数据协调核心
2. **客户端实现**: `src/mooncake_client/real_client.cpp` — 读写路径
3. **分段管理**: `src/mooncake_master/segment_manager.cpp` — 存储空间分配
4. **驱逐策略**: `src/mooncake_master/eviction_manager.cpp` — 淘汰逻辑
5. **租约管理**: `src/mooncake_master/lease_manager.cpp` — TTL 管理
6. **三级存储**: `src/tiered_storage.cpp` — 数据在 G1/G2/G3 间迁移

## 优化维度

### 1. 元数据服务 HA 与一致性
- **代码入口**: `master_service.cpp`, `metadata_store.cpp`
- **当前状态**: 支持 HTTP/ETCD/Redis 三种 metadata 后端，1024 分片
- **关键问题**:
  - Master 是单点还是支持多副本？故障转移时间？
  - 元数据的一致性模型？（强一致 vs. 最终一致）
  - 分段分配是否需要分布式锁？
- **Domain Knowledge 映射**:
  - `algorithms/distributed-consensus/KNOWLEDGE.md` — 共识协议、leader 选举
  - `architecture/cloud-native/KNOWLEDGE.md` — 云原生服务架构

### 2. 驱逐策略
- **代码入口**: `eviction_manager.cpp`
- **当前状态**: 基于 TTL（hard 5s, soft 30min）+ 容量驱逐
- **关键问题**:
  - 仅 TTL 是否足够？有没有基于访问频率的策略？
  - 驱逐的粒度是整对象还是部分？
  - 驱逐决策是中心化还是分布式？
- **Domain Knowledge 映射**:
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 缓存替换算法
  - `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 层次化存储管理

### 3. 三级存储迁移
- **代码入口**: `tiered_storage.cpp`
- **当前状态**: G1 (GPU HBM) → G2 (CPU DRAM) → G3 (NVMe SSD)
- **关键问题**:
  - 迁移触发条件？（容量阈值？访问频率？）
  - 迁移是否阻塞读写？
  - 降级（G1→G2）的延迟 vs. 升级（G3→G2）的策略
- **Domain Knowledge 映射**:
  - `performance/storage-filesystem/KNOWLEDGE.md` — 分级存储
  - `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 存储层次

### 4. 副本放置策略
- **代码入口**: `replica_manager.cpp`
- **当前状态**: 支持多副本
- **关键问题**:
  - 副本数如何选择？写入时全量同步还是异步？
  - 副本放置是否感知拓扑？（同 rack? 跨 DC?）
  - 读写 quorum 配置？
- **Domain Knowledge 映射**:
  - `architecture/cloud-native/KNOWLEDGE.md` — 副本策略
  - `algorithms/distributed-consensus/KNOWLEDGE.md` — 复制协议

### 5. 分段分配与碎片化
- **代码入口**: `segment_manager.cpp`
- **当前状态**: 分段 (segment) 是存储的基本单元
- **关键问题**:
  - 分段大小是否固定？如何选择最优值？
  - 大量小对象是否导致碎片化？
  - 分段回收和合并策略？
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — 内存/存储分配
  - `performance/storage-filesystem/KNOWLEDGE.md` — 空间管理

### 6. 批量操作优化
- **代码入口**: `real_client.cpp:batch_put_from()` / `batch_get_into()`
- **当前状态**: 支持批量 put/get
- **关键问题**:
  - 批量操作的内部调度？（全部并发？限流？）
  - 批量失败时的行为？（全部回滚？部分成功？）
  - 批量大小是否有上限？
- **Domain Knowledge 映射**:
  - `performance/gpu-ai-performance/KNOWLEDGE.md` — 批量推理
  - `performance/optimization-paradigms/KNOWLEDGE.md` — 批处理模式

### 7. 客户端连接池
- **代码入口**: `real_client.cpp:setup()`
- **当前状态**: 每个 client 建立一个连接
- **关键问题**:
  - 连接建立的开销？
  - 是否支持连接复用和多路复用？
  - Master 的连接数上限和负载？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 连接管理
  - `performance/concurrency/KNOWLEDGE.md` — 并发模型

## 分析工作流

同 Transfer Engine 的分析流程。详见 `common/SKILL.md`。
