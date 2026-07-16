# Mooncake Source Tree → Component Map

本文档维护 Mooncake 源代码树到各组件的映射关系。当 Mooncake 仓库结构变化时需同步更新。

> **最后更新**: 2026-07-16
> **变更**: 新增子组件映射 — transfer-engine 拆为 transport/tent/memory/topology，store 拆为 storage-backend/master/client/replication
> **基于 Mooncake commit**: 当前 main 分支

---

## Transfer Engine / TENT

> **子组件映射**: transport / tent / memory / topology（详见各子目录 SKILL.md）

### 核心源代码
```
mooncake-transfer-engine/
├── include/
│   ├── transfer_engine.h              # TransferEngine 主接口
│   ├── transfer_engine_c.h            # C FFI
│   ├── cuda_alike.h                   # 跨 GPU 厂商统一抽象
│   ├── common.h                       # 公共类型定义
│   ├── transport/                     # [transport] 传输协议抽象接口
│   │   └── transport.h                # Transport 基类
│   └── tent/                          # [tent] TENT 运行时 API
│       ├── tent_runtime.h
│       └── tent_config.h
├── src/
│   ├── transfer_engine.cpp            # TransferEngine 核心实现
│   ├── transfer_metadata*.cpp/h       # 传输元数据管理
│   ├── transport/                     # [transport] 各协议实现
│   │   ├── rdma_transport.cpp         #   RDMA (IB/RoCE/eRDMA)
│   │   ├── tcp_transport.cpp          #   TCP
│   │   ├── nvlink_transport.cpp       #   NVLink
│   │   ├── efa_transport.cpp          #   AWS EFA
│   │   ├── cxl_transport.cpp          #   CXL
│   │   ├── nvmeof_transport.cpp       #   NVMe-oF
│   │   ├── ascend_transport.cpp       #   Ascend Direct
│   │   └── shm_transport.cpp          #   共享内存
│   ├── memory_location.cpp            # [memory] 内存位置抽象
│   ├── topology.cpp                   # [topology] GPU-NIC 拓扑发现
│   └── multi_transport.cpp            # [transport] 多协议混合
├── tent/                              # [tent] TENT 下一代运行时
│   ├── runtime/
│   │   ├── slice_scheduler.cpp        #   切片调度器
│   │   ├── transport_selector.cpp     #   动态协议选择
│   │   └── telemetry_collector.cpp    #   遥测收集
│   ├── config/
│   │   └── tent_config.yaml           #   配置模板
│   └── benchmark/
│       └── tent_bench.cpp             #   基准测试
└── nvlink-allocator/                  # [memory] NVLink 内存分配器
    ├── nvlink_allocator.h
    └── nvlink_allocator.cpp
```

### 关键类/函数
| 符号 | 文件 | 说明 |
|------|------|------|
| `TransferEngine::init()` | `transfer_engine.cpp` | 初始化引擎，连接 metadata server |
| `TransferEngine::registerLocalMemory()` | `transfer_engine.cpp` | 注册本地内存（pin memory for RDMA） |
| `TransferEngine::submitTransfer()` | `transfer_engine.cpp` | 提交批量传输请求 |
| `TransferEngine::getTransferStatus()` | `transfer_engine.cpp` | 查询传输状态 |
| `Transport` (基类) | `include/transport/transport.h` | 传输协议抽象接口 |
| `RdmaTransport` | `src/transport/rdma_transport.cpp` | RDMA 协议实现 |
| `SliceScheduler` | `tent/runtime/slice_scheduler.cpp` | TENT 多路径切片调度 |
| `TransportSelector` | `tent/runtime/transport_selector.cpp` | TENT 动态传输选择 |

### 构建配置
- `CMakeLists.txt` 中的 `USE_CUDA`, `USE_RDMA`, `USE_TCP`, `USE_TENT` 等 flag
- 输出: `libtransfer_engine.so`

---

## Mooncake Store

> **子组件映射**: storage-backend / master / client / replication（详见各子目录 SKILL.md）

### 核心源代码
```
mooncake-store/
├── include/
│   ├── real_client.h                  # [client] RealClient 接口
│   ├── store_c.h                      # [client] C FFI
│   ├── p2p_store.h                    # [client] P2P Store 接口
│   └── types.h                        # 公共数据结构
├── src/
│   ├── mooncake_master/               # [master] Master 元数据协调服务
│   │   ├── master_service.cpp         #   Master 主逻辑
│   │   ├── segment_manager.cpp        #   分段管理器
│   │   ├── eviction_manager.cpp       #   驱逐管理器
│   │   ├── replica_manager.cpp        # [replication] 副本管理器
│   │   ├── lease_manager.cpp          #   租约管理器
│   │   └── metadata_store.cpp         #   元数据持久化
│   ├── mooncake_client/               # [client] 客户端实现
│   │   ├── real_client.cpp            #   RealClient (RDMA + 内存管理)
│   │   ├── dummy_client.cpp           #   DummyClient (RPC 代理)
│   │   └── client_allocator.cpp       # [storage-backend] 客户端侧内存分配
│   ├── p2p_store.cpp                  # [client] P2P Store
│   ├── storage_backend.cpp            # [storage-backend] DRAM/SSD 后端
│   ├── tiered_storage.cpp             # [storage-backend] 三级存储 (G1/G2/G3)
│   └── transfer_task_scheduler.cpp    # [client] 传输任务调度
├── tools/
│   ├── mc_store_rest_server.cpp       # REST API 服务器
│   └── store_bench.cpp                # Store 基准测试
└── cmake/
    └── store.cmake
```

### 关键类/函数
| 符号 | 文件 | 说明 |
|------|------|------|
| `RealClient::setup()` | `real_client.cpp` | 客户端初始化，连接 Master |
| `RealClient::put()` / `RealClient::get()` | `real_client.cpp` | 同步读/写对象 |
| `RealClient::put_from()` / `RealClient::get_into()` | `real_client.cpp` | 零拷贝读/写（预注册 buffer） |
| `RealClient::batch_put_from()` / `RealClient::batch_get_into()` | `real_client.cpp` | 批量操作 |
| `MasterService` | `master_service.cpp` | Master 服务主逻辑 |
| `SegmentManager` | `segment_manager.cpp` | 分段分配与回收 |
| `EvictionManager` | `eviction_manager.cpp` | 基于 TTL 和容量的驱逐策略 |
| `LeaseManager` | `lease_manager.cpp` | 租约管理（hard 5s / soft 30min） |
| `TieredStorage` | `tiered_storage.cpp` | G1→G2→G3 数据迁移 |

### 构建配置
- `CMakeLists.txt` 中的 `WITH_STORE`, `USE_HTTP`, `USE_ETCD`, `USE_REDIS` flag
- 输出: `libmooncake_store.so`, `mooncake_master`, `mooncake_client`

---

## Conductor (KV Cache Indexer)

### 核心源代码
```
docs/design/conductor/
├── conductor-architecture-design.md   # Conductor 架构设计文档
└── indexer-api-design.md              # Indexer API 设计

(注: Conductor 的核心代码可能位于独立仓库或尚未完全开源在 Mooncake 主仓库中。
 当前以设计文档为主进行分析。)
```

### 关键概念
| 组件 | 说明 |
|------|------|
| `EventManager` | 订阅 KV cache 事件（Put/Evict/Update） |
| `ZMQClient` | 与 Mooncake Store 的 ZMQ 通信客户端 |
| `KVEventHandler` | KV 事件处理逻辑 |
| `PrefixCacheTable` | 前缀缓存表（XXH3-64 滚动哈希） |
| `ModelContext` | 模型上下文 = (tenant_id, model_name, lora_name, block_size, salt, instance_id) |

### HTTP API
| 端点 | 方法 | 说明 |
|------|------|------|
| `/register` | POST | 注册 KV 事件发布者 |
| `/unregister` | POST | 移除发布者 |
| `/query` | POST | 按 token IDs 查询缓存命中 |
| `/query_by_hash` | POST | 按 XXH3-64 哈希查询缓存命中 |

---

## Framework Integrations

### 核心源代码
```
mooncake-integration/
├── CMakeLists.txt                     # pybind11 构建配置
├── engine.cpp                         # -> engine.so (TransferEngine Python binding)
├── store.cpp                          # -> store.so (MooncakeDistributedStore Python binding)
├── async_store.py                     # 异步 Store 辅助类
├── allocator.py                       # 内存分配器 (Python)
├── allocator_ascend_npu.py            # 华为 Ascend NPU 分配器
├── mooncake_config.py                 # 配置辅助类 (环境变量解析)
└── ep_pg_staging/                     # Expert Parallelism / Process Group (staging)

mooncake-wheel/
├── pyproject.toml                     # Python 包配置
└── mooncake/
    └── __init__.py                    # 包入口

docs/source/getting_started/examples/
├── sglang-integration-v1.md           # SGLang HiCache 集成 (L3 backend)
├── lmcache-integration.md             # LMCache 集成
└── vllm-integration/
    ├── disagg-prefill-decode.md        # vLLM PD 分离部署
    ├── kv-cache-storage.md            # vLLM KV Cache 存储
    └── vllmv1-lmcache-integration.md   # vLLM v1 + LMCache 集成
```

### 关键类/函数
| 符号 | 文件 | 说明 |
|------|------|------|
| `TransferEngine` (Python) | `engine.cpp` | Python 侧 Transfer Engine 绑定 |
| `MooncakeDistributedStore` (Python) | `store.cpp` | Python 侧 Store 绑定 |
| `AsyncStore` | `async_store.py` | 异步操作包装器 |
| `Allocator` | `allocator.py` | Python 侧内存分配器 |

### 框架连接器（在各自框架仓库中）
| 框架 | 连接器 | 仓库位置 |
|------|--------|---------|
| vLLM | `MooncakeConnector` (KV Connector V1) | vLLM 主仓库 |
| vLLM | `MooncakeStoreConnector` | vLLM 主仓库 |
| SGLang | HiCache L3 Backend | SGLang 主仓库 |
| TensorRT-LLM | `mooncake_utils` | TensorRT-LLM 仓库 |
| LMDeploy | PD Backend Plugin | LMDeploy 仓库 |

---

## Build & Deploy

### 构建系统
```
CMakeLists.txt                         # 顶层 CMake (包含所有子项目)
mooncake-common/
├── common.cmake                       # 全局编译选项和 feature flags
└── src/                               # libmooncake_common.so
dependencies.sh                        # 依赖安装脚本
scripts/
├── build_wheel.sh                     # Python wheel 打包
└── ci_install.sh                      # CI 依赖安装
.github/workflows/                     # GitHub Actions CI/CD
extern/
├── yalantinglibs/                     # 协程 RPC 框架 (git submodule)
└── pybind11/                          # Python 绑定 (git submodule)
```

### 关键 CMake Flags
| Flag | 默认值 | 影响组件 |
|------|--------|---------|
| `WITH_TE` | ON | 构建 Transfer Engine |
| `WITH_STORE` | ON | 构建 Mooncake Store |
| `WITH_EP` | OFF | 构建 Expert Parallelism Python 扩展 |
| `USE_CUDA` | OFF | NVIDIA GPU 支持 |
| `USE_HIP` | OFF | AMD GPU 支持 |
| `USE_TCP` | ON | TCP 传输 |
| `USE_TENT` | OFF | TENT 下一代引擎 |
| `USE_HTTP` | ON | HTTP 元数据服务 |
| `USE_ETCD` | OFF | etcd 元数据后端 |
| `USE_REDIS` | OFF | Redis 元数据后端 |

### 构建产物
| 产物 | 类型 | 来源 |
|------|------|------|
| `libtransfer_engine.so` | 共享库 | `mooncake-transfer-engine/` |
| `libmooncake_store.so` | 共享库 | `mooncake-store/` |
| `libmooncake_common.so` | 共享库 | `mooncake-common/` |
| `engine.so` | Python 扩展 | `mooncake-integration/` |
| `store.so` | Python 扩展 | `mooncake-integration/` |
| `mooncake_master` | 可执行文件 | `mooncake-store/` |
| `mooncake_client` | 可执行文件 | `mooncake-store/` |
| `transfer_engine_bench` | 可执行文件 | `mooncake-transfer-engine/` |

---

## 文档资源

```
docs/source/
├── index.md                           # 文档首页
├── getting_started/
│   ├── build.md                       # 构建指南
│   ├── quick-start.md                 # 快速开始
│   └── supported-protocols.md        # 支持的传输协议列表
└── design/
    ├── mooncake-store.md              # Store 设计文档
    ├── hicache-design.md              # HiCache 设计
    └── tent/
        └── overview.md                # TENT 总览
```

---

## 变更记录

| 日期 | 变更 |
|------|------|
| 2026-07-14 | 初始版本，基于 Mooncake main 分支 |
