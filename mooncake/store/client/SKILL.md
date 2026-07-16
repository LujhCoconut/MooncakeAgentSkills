# Mooncake Store — Client 数据路径优化

Client 是 Mooncake Store 的数据平面入口，负责处理 put/get 请求、管理连接、批量操作和 P2P 共享。分为 RealClient（直接 RDMA + 内存管理）和 DummyClient（RPC 代理）两种模式。

## 源代码地图

| 文件 | 说明 |
|------|------|
| `mooncake-store/src/mooncake_client/real_client.cpp` | RealClient 实现 (RDMA + 内存直管) |
| `mooncake-store/src/mooncake_client/dummy_client.cpp` | DummyClient 实现 (RPC 代理) |
| `mooncake-store/src/mooncake_client/client_allocator.cpp` | 客户端侧内存分配器 |
| `mooncake-store/include/real_client.h` | RealClient 接口 |
| `mooncake-store/include/store_c.h` | C FFI 接口 |
| `mooncake-store/src/p2p_store.cpp` | P2P Store 实现 |
| `mooncake-store/src/transfer_task_scheduler.cpp` | 传输任务调度器 |

## 优化维度

### 1. 读写路径优化
- **代码入口**: `real_client.cpp:put()` / `get()` / `put_from()` / `get_into()`
- **当前状态**: 同步 put/get + 零拷贝 put_from/get_into
- **关键问题**:
  - 小对象的读写延迟？（metadata 查询开销 > 数据传输开销？）
  - put_from/get_into 的零拷贝路径是否真的零拷贝？（验证整个调用链）
  - 读写是否支持 pipelining？（多个请求复用同一连接）
  - 读写的内存屏障/ fencing？（GPU 侧的数据可见性）
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — I/O 路径优化
  - `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU 数据搬运

### 2. 连接管理与复用
- **代码入口**: `real_client.cpp:setup()`
- **当前状态**: 每个 client 独立建立连接
- **关键问题**:
  - 连接建立的开销？（RDMA QP 创建、内存注册等）
  - 是否支持连接池和多路复用？
  - 连接的心跳和保活？
  - 大量 client 时的 Master 连接数压力？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 连接管理
  - `performance/concurrency/KNOWLEDGE.md` — 并发连接模型

### 3. 批量操作优化
- **代码入口**: `real_client.cpp:batch_put_from()` / `batch_get_into()`
- **当前状态**: 支持批量 put/get
- **关键问题**:
  - 批量操作的内部调度策略？（全部并发？有限并发？流水线？）
  - 批量失败的部分成功语义？（全部回滚 vs. 部分成功 vs. best-effort）
  - 批量大小上限和自适应？
  - 批量操作的排序？（是否可乱序执行以最大化吞吐？）
- **Domain Knowledge 映射**:
  - `performance/optimization-paradigms/KNOWLEDGE.md` — 批处理模式
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 批量调度

### 4. RealClient vs. DummyClient
- **代码入口**: `real_client.cpp`, `dummy_client.cpp`
- **当前状态**: 两种模式，DummyClient 通过 RPC 代理到 RealClient
- **关键问题**:
  - DummyClient 的 RPC 开销？（适合哪些场景？）
  - DummyClient→RealClient 的代理是否单点？
  - 两种模式的切换是否透明？
- **Domain Knowledge 映射**:
  - `architecture/cloud-native/KNOWLEDGE.md` — 代理模式

### 5. Client 可观测性
- **关键指标**:
  - put/get 延迟 (p50/p99/p999)
  - 批量操作的吞吐 (ops/s) 和带宽 (GB/s)
  - 连接数、连接建立/断开速率
  - 错误率（按错误类型分类）
  - 各操作的数据量分布
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
