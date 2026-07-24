---
name: api-reference
description: Mooncake Python API 参考指南。覆盖 Mooncake Store (分布式 KV Cache)、Transfer Engine (RDMA/TCP 传输)、Mooncake EP/Backend (Expert Parallelism) 的完整 API 用法、代码示例、最佳实践。
argument-hint: "[store|engine|ep|env|pattern] <查询内容>"
---

# Mooncake Python API 参考

使用本 skill 帮助用户使用 Mooncake Python API 进行分布式存储和高性能数据传输。

> **由AI总结，仅供参考！** 本指南基于 Mooncake 源码自动生成，API 签名可能因版本而异。以 `kvcache-ai/Mooncake` 仓库最新版为准。

## 触发条件

当用户询问以下内容时使用本 skill：
- Mooncake Store 分布式 KV cache 存储
- Transfer Engine RDMA/TCP 数据传输
- Mooncake 服务启动（master、metadata server）
- PyTorch tensor 在 Store 中的存取
- 零拷贝操作与 buffer 管理
- 批量操作与副本配置
- Mooncake EP (Expert Parallelism) 与 Mooncake Backend
- Mooncake Python API 排错

## 路由指南

- 大多数 vLLM / SGLang 用户：从 `docs/source/getting_started/quick-start.md` 开始
- PD 分离场景：引导到 Quick Start 中的 SGLang/vLLM 集成指南
- Store 集成：引导到 Quick Start 中的 Store setup 指南。不要在通用 API 回答中重复 `mooncake_master` 启动命令
- 底层 Transfer Engine 直用：使用 `docs/source/design/transfer-engine/index.md#using-transfer-engine-in-your-projects`
- API 签名与方法详情：使用 `docs/source/python-api-reference/` 下的文档

## 关联子命令

| 场景 | 使用 |
|------|------|
| 想深入了解 API 背后的 C++ 实现 | `/mooncake-agent-skills optimize "..."` |
| API 调用报错需要排查 | `/mooncake-agent-skills troubleshoot` |
| 环境变量/配置概念问题 | `/mooncake-agent-skills qa "..."` |

---

## 核心组件

### 1. Mooncake Store (分布式 KV Cache)

**Import:**
```python
from mooncake.store import MooncakeDistributedStore, ReplicateConfig
```

**Basic Setup:**
```python
store = MooncakeDistributedStore()
store.setup(
    "localhost",                          # local_hostname
    "http://localhost:8080/metadata",     # metadata_server
    512*1024*1024,                        # global_segment_size (512MB)
    128*1024*1024,                        # local_buffer_size (128MB)
    "tcp",                                # protocol ("tcp" or "rdma")
    "",                                   # rdma_devices (empty for auto-select)
    "localhost:50051"                     # master_server_address
)
```

> **性能提示**：`global_segment_size` 决定了 Store 的总内存池大小。若报 `NO_AVAILABLE_HANDLE` (-200)，增大此值或用 `/mooncake-agent-skills optimize "Store 内存池驱逐策略"` 分析驱逐优化。

**Common Operations:**
```python
# Put/Get
store.put("key", b"value")
data = store.get("key")

# Batch operations (推荐: 比逐条 put/get 吞吐高 3-10×)
store.put_batch(["key1", "key2"], [b"val1", b"val2"])
values = store.get_batch(["key1", "key2"])

# Check existence
exists = store.is_exist("key")  # Returns 1 (exists), 0 (not exists), -1 (error)

# Remove
store.remove("key")
store.remove_by_regex("^prefix_.*")
store.remove_all()

# Cleanup
store.close()
```

**Zero-Copy Operations (Advanced):**
```python
import numpy as np

# Create and register buffer
buffer = np.zeros(100*1024*1024, dtype=np.uint8)
buffer_ptr = buffer.ctypes.data
store.register_buffer(buffer_ptr, buffer.nbytes)

# Zero-copy put
store.put_from("key", buffer_ptr, buffer.nbytes)

# Zero-copy get
recv_buffer = np.empty(100*1024*1024, dtype=np.uint8)
recv_ptr = recv_buffer.ctypes.data
store.register_buffer(recv_ptr, recv_buffer.nbytes)
bytes_read = store.get_into("key", recv_ptr, recv_buffer.nbytes)

# Cleanup
store.unregister_buffer(buffer_ptr)
store.unregister_buffer(recv_ptr)
```

> **零拷贝前提**：buffer 必须先 `register_buffer` 才能用于 `put_from`/`get_into`。RDMA 模式下的零拷贝需要 GPU-NIC 拓扑亲和（见 `mooncake/transfer-engine/topology/`）。

**PyTorch Tensor Operations:**
```python
import torch

# Simple tensor operations
tensor = torch.randn(100, 100)
store.put_tensor("my_tensor", tensor)
retrieved = store.get_tensor("my_tensor")

# Batch tensor operations
tensors = [torch.randn(100, 100) for _ in range(3)]
store.batch_put_tensor(["t1", "t2", "t3"], tensors)
retrieved_tensors = store.batch_get_tensor(["t1", "t2", "t3"])

# Tensor Parallelism (TP) support
store.put_tensor_with_tp("model_weights", tensor, tp_rank=0, tp_size=4, split_dim=0)
shard = store.get_tensor_with_tp("model_weights", tp_rank=0, tp_size=4)
```

**Replication Configuration:**
```python
config = ReplicateConfig()
config.replica_num = 3              # Number of replicas
config.with_soft_pin = True         # Keep in memory longer (减少驱逐概率)
config.preferred_segment = "host:port"  # Preferred location

store.put("key", b"value", config)
```

> `with_soft_pin = True` 会降低该对象的驱逐优先级，但不保证永不被驱逐。需要强保证请增大 `global_segment_size`。

### 2. Transfer Engine (高性能数据传输)

**Import:**
```python
from mooncake.engine import TransferEngine, TransferOpcode, TransferNotify
```

**Basic Setup:**
```python
engine = TransferEngine()
engine.initialize(
    "127.0.0.1:12345",      # local_hostname
    "127.0.0.1:2379",       # metadata_server (or "etcd://...")
    "tcp",                  # protocol ("tcp" or "rdma")
    ""                      # device_name (empty for all devices)
)
```

> **协议选择**：开发/测试用 TCP，生产用 RDMA。想分析 RDMA 延迟瓶颈 → `/mooncake-agent-skills optimize "降低 RDMA 传输延迟"`

**Buffer Management:**
```python
# Allocate managed buffer
buffer_size = 1024 * 1024  # 1MB
buffer_addr = engine.allocate_managed_buffer(buffer_size)

# Write/read bytes
data = b"Hello, Transfer Engine!"
engine.write_bytes_to_buffer(buffer_addr, data, len(data))
read_data = engine.read_bytes_from_buffer(buffer_addr, len(data))

# Free buffer
engine.free_managed_buffer(buffer_addr, buffer_size)
```

**Data Transfer Operations:**
```python
# Synchronous write
result = engine.transfer_sync_write(
    "target_host:port",     # target_hostname
    local_buffer_addr,      # buffer
    remote_buffer_addr,     # peer_buffer_address
    data_length             # length
)

# Synchronous read
result = engine.transfer_sync_read(
    "target_host:port",
    local_buffer_addr,
    remote_buffer_addr,
    data_length
)

# Asynchronous write (适合流水线)
batch_id = engine.transfer_submit_write(
    "target_host:port",
    local_buffer_addr,
    remote_buffer_addr,
    data_length
)

# Check status
status = engine.transfer_check_status(batch_id)
# Returns: 1 (completed), 0 (in progress), -1 (failed), -2 (timeout)
```

> **异步 vs 同步**：高吞吐场景优先用 `transfer_submit_write` + `transfer_check_status` 做流水线，避免同步等待浪费带宽。

**Batch Transfer Operations:**
```python
# Batch synchronous write
local_addrs = [addr1, addr2, addr3]
remote_addrs = [remote1, remote2, remote3]
lengths = [len1, len2, len3]

result = engine.batch_transfer_sync_write(
    "target_host:port",
    local_addrs,
    remote_addrs,
    lengths
)

# Batch asynchronous operations
batch_id = engine.batch_transfer_async_write(
    "target_host:port",
    local_addrs,
    remote_addrs,
    lengths
)

# Wait for completion
result = engine.get_batch_transfer_status([batch_id])
```

> **批量传输优化**：TENT 的切片调度会将大传输拆分为多条路径并行传输。切片大小由 `MC_BATCH_SIZE` 控制（默认值见 `qa/KNOWLEDGE.md` 配置参数表）。

**Memory Registration (for RDMA):**
```python
import numpy as np

buffer = np.ones(1024*1024, dtype=np.uint8)
buffer_ptr = buffer.ctypes.data
buffer_size = buffer.nbytes

# Register memory
engine.register_memory(buffer_ptr, buffer_size)

# Use buffer for transfers...

# Unregister when done
engine.unregister_memory(buffer_ptr)
```

### 3. Mooncake EP & Backend (Expert Parallelism)

**Mooncake Backend (Fault-Tolerant Collectives):**
```python
import torch
import torch.distributed as dist
from mooncake import pg

# Initialize with fault tolerance
active_ranks = torch.ones((world_size,), dtype=torch.int32, device="cuda")
dist.init_process_group(
    backend="mooncake",
    rank=rank,
    world_size=world_size,
    pg_options=pg.MooncakeBackendOptions(active_ranks),
)

# Use standard PyTorch distributed APIs
dist.all_gather(...)
dist.all_reduce(...)

# Check for failures
assert active_ranks.all()  # Verify no ranks are broken
```

**Mooncake EP (Expert Parallelism):**
```python
from mooncake.mooncake_ep_buffer import Buffer
import torch.distributed as dist

# Calculate buffer size
num_ep_buffer_bytes = Buffer.get_ep_buffer_size_hint(
    num_max_dispatch_tokens_per_rank=1024,
    hidden=4096,
    num_ranks=8,
    num_experts=64
)

# Create buffer (must be Mooncake Backend process group)
buffer = Buffer(group=dist.group.WORLD, num_ep_buffer_bytes=num_ep_buffer_bytes)

# Dispatch/combine operations
active_ranks = torch.ones((num_ranks,), dtype=torch.int32, device="cuda")
buffer.dispatch(..., active_ranks=active_ranks, timeout_us=1000000)
buffer.combine(..., active_ranks=active_ranks, timeout_us=1000000)
```

---

## 启动服务

### 启动 Master（含 HTTP metadata server）
```bash
mooncake_master \
  --enable_http_metadata_server=true \
  --http_metadata_server_host=0.0.0.0 \
  --http_metadata_server_port=8080 \
  --default_kv_lease_ttl=5000
```

### 使用外部 etcd（生产环境推荐）
```bash
# Start etcd
etcd --listen-client-urls http://0.0.0.0:2379 \
     --advertise-client-urls http://0.0.0.0:2379

# Start master
mooncake_master --default_kv_lease_ttl=5000
```

---

## 环境变量

### Transfer Engine
| 变量 | 说明 | 默认值 |
|------|------|--------|
| `MC_METADATA_SERVER` | Metadata server URL | 无（必填） |
| `MC_FORCE_TCP` | 强制 TCP 传输（设 "true"） | false |
| `MC_LOG_LEVEL` | 日志级别 (0=INFO, 1=WARNING, 2=ERROR) | 1 |
| `MC_MS_AUTO_DISC` | RDMA 设备自动发现（设 "1"） | 1 |
| `MC_MS_FILTERS` | 过滤 RDMA 设备 (如 "mlx5_0,mlx5_2") | 空 |
| `MC_TRANSFER_TIMEOUT` | 传输超时（秒） | 30 |
| `MC_GID_INDEX` | RDMA GID index | 默认 |
| `MC_MTU` | RDMA MTU 大小 | 默认 |
| `MC_ENABLE_DEST_DEVICE_AFFINITY` | 减少 QP 创建（修 "Failed to create QP"） | 0 |

### Mooncake Store
| 变量 | 说明 | 默认值 |
|------|------|--------|
| `MC_STORE_CLUSTER_ID` | 集群标识 | "mooncake" |
| `MC_STORE_USE_HUGEPAGE` | 启用 hugepage | false |
| `MC_STORE_MEMCPY` | 本地 memcpy 优化（设 "1"） | 0 |
| `MC_STORE_CLIENT_METRIC` | 客户端指标 | 1 |
| `MC_YLT_LOG_LEVEL` | yalantinglibs 日志级别 | info |

---

## 常见模式

### Pattern 1: 简单 KV Store
```python
from mooncake.store import MooncakeDistributedStore

store = MooncakeDistributedStore()
store.setup("localhost", "http://localhost:8080/metadata",
            512*1024*1024, 128*1024*1024, "tcp", "", "localhost:50051")

store.put("config", b'{"model": "llama-7b"}')
config = store.get("config")

store.close()
```

### Pattern 2: 高性能 Tensor 存储（含副本）
```python
import torch
from mooncake.store import MooncakeDistributedStore, ReplicateConfig

store = MooncakeDistributedStore()
store.setup("localhost", "http://localhost:8080/metadata",
            512*1024*1024, 128*1024*1024, "rdma", "mlx5_0", "localhost:50051")

config = ReplicateConfig()
config.replica_num = 2
config.with_soft_pin = True

tensor = torch.randn(1000, 1000)
store.put_tensor("weights", tensor, config)

retrieved = store.get_tensor("weights")
store.close()
```

### Pattern 3: 零拷贝批量操作
```python
import numpy as np
from mooncake.store import MooncakeDistributedStore

store = MooncakeDistributedStore()
store.setup("localhost", "http://localhost:8080/metadata",
            512*1024*1024, 16*1024*1024, "rdma", "", "localhost:50051")

num_buffers = 10
buffers = [np.random.randn(1024*1024).astype(np.float32) for _ in range(num_buffers)]
buffer_ptrs = [buf.ctypes.data for buf in buffers]
sizes = [buf.nbytes for buf in buffers]

for ptr, size in zip(buffer_ptrs, sizes):
    store.register_buffer(ptr, size)

keys = [f"tensor_{i}" for i in range(num_buffers)]
store.batch_put_from(keys, buffer_ptrs, sizes)

recv_buffers = [np.empty(1024*1024, dtype=np.float32) for _ in range(num_buffers)]
recv_ptrs = [buf.ctypes.data for buf in recv_buffers]

for ptr, size in zip(recv_ptrs, sizes):
    store.register_buffer(ptr, size)

store.batch_get_into(keys, recv_ptrs, sizes)

for ptr in buffer_ptrs + recv_ptrs:
    store.unregister_buffer(ptr)

store.close()
```

### Pattern 4: Transfer Engine 直传
```python
from mooncake.engine import TransferEngine
import numpy as np

engine = TransferEngine()
engine.initialize("127.0.0.1:12345", "127.0.0.1:2379", "tcp", "")

buffer = np.ones(1024*1024, dtype=np.uint8)
buffer_ptr = buffer.ctypes.data
engine.register_memory(buffer_ptr, buffer.nbytes)

remote_addr = engine.get_first_buffer_address("target_host:port")

data = b"Hello from Transfer Engine!"
engine.write_bytes_to_buffer(buffer_ptr, data, len(data))
result = engine.transfer_sync_write("target_host:port", buffer_ptr, remote_addr, len(data))

engine.unregister_memory(buffer_ptr)
```

---

## 错误处理

所有方法返回状态码：
- `0`: 成功
- 负值: 错误码

常见检查：
```python
# Store 操作
result = store.put("key", b"value")
if result != 0:
    print(f"Put failed with error code: {result}")

# 存在性检查
exists = store.is_exist("key")
if exists == 1:
    print("Key exists")
elif exists == 0:
    print("Key not found")
else:
    print("Error checking existence")

# Transfer 操作
result = engine.transfer_sync_write(...)
if result == 0:
    print("Transfer successful")
else:
    print(f"Transfer failed with code: {result}")
```

> **错误码速查**：完整错误码表见 `/mooncake-agent-skills troubleshoot` → Error Code Quick Reference。也可用 `qa` 查询特定错误。

---

## API → 源码映射

> 以下映射帮助你从 Python API 追溯到 C++ 实现，用于深度优化或调试。

### Mooncake Store

| Python 方法 | C++ 实现（关键文件） |
|------------|---------------------|
| `MooncakeDistributedStore()` | `mooncake-store/include/client.h` → `DistributedStore` |
| `store.setup()` | `mooncake-store/src/client.cpp` → `DistributedStore::setup()` |
| `store.put()` / `store.get()` | `mooncake-store/src/client.cpp` → `put()` / `get()` |
| `store.put_batch()` / `store.get_batch()` | `mooncake-store/src/client.cpp` → batch variants |
| `store.put_from()` / `store.get_into()` | `mooncake-store/src/client.cpp` → zero-copy path, 经 `transfer_engine` |
| `store.put_tensor()` | `mooncake-store/src/client.cpp` → tensor serialization → `put()` |
| `store.register_buffer()` | `mooncake-store/src/client.cpp` → `register_buffer()` → Transfer Engine memory registration |
| `store.is_exist()` | `mooncake-store/src/client.cpp` → RPC → master metadata lookup |
| `ReplicateConfig` | `mooncake-store/include/replicate_config.h` |

### Transfer Engine

| Python 方法 | C++ 实现（关键文件） |
|------------|---------------------|
| `TransferEngine()` | `mooncake-transfer-engine/include/transfer_engine.h` → `TransferEngine` |
| `engine.initialize()` | `mooncake-transfer-engine/src/transfer_engine.cpp` → `init()` |
| `engine.allocate_managed_buffer()` | `mooncake-transfer-engine/src/transfer_engine.cpp` → memory allocator |
| `engine.transfer_sync_write()` | `mooncake-transfer-engine/src/transfer_engine.cpp` → TENT 调度 → RDMA/TCP transport |
| `engine.transfer_submit_write()` | 同上，异步路径 → batch_id 管理 |
| `engine.register_memory()` | `mooncake-transfer-engine/src/transfer_engine.cpp` → RDMA `ibv_reg_mr()` |
| `engine.batch_transfer_async_write()` | TENT multi-path slice → 并行 RDMA writes |

### Mooncake EP / Backend

| Python 方法 | C++ 实现（关键文件） |
|------------|---------------------|
| `MooncakeBackendOptions` | `mooncake-pg/` → PyTorch distributed backend plugin |
| `Buffer.get_ep_buffer_size_hint()` | `mooncake-ep/` → buffer sizing logic |
| `buffer.dispatch()` / `buffer.combine()` | `mooncake-ep/` → NCCL/RDMA dispatch/combine kernel |

---

## 故障排查速查

| 症状 | 可能原因 | 解决 |
|------|---------|------|
| `put()` 返回 -200 | `NO_AVAILABLE_HANDLE` 内存池耗尽 | 增大 `global_segment_size` 或分析驱逐策略 |
| `transfer_sync_write()` 返回 -14 | RDMA 设备未找到 | 检查 `ibv_devices`，确认设备名 |
| `initialize()` 连接失败 | metadata server 不可达 | `/mooncake-agent-skills troubleshoot` |
| `register_buffer()` 失败 | 内存注册超限 | `ulimit -l unlimited` |
| `store.get()` 返回空 | key 不存在 / lease 过期 | 用 `is_exist()` 检查，增大 `default_kv_lease_ttl` |

> 深度排查：`/mooncake-agent-skills troubleshoot` 提供 8 步系统化诊断流程。

---

## 最佳实践

1. **Always close stores**: 用完调用 `store.close()`
2. **Register buffers for zero-copy**: RDMA 操作必须先注册内存
3. **Use batch operations**: 批量操作吞吐显著高于逐条（3-10×）
4. **Configure replication**: 重要数据用 `ReplicateConfig` 配置副本
5. **Use soft pinning**: 频繁访问的对象设置 `with_soft_pin = True`
6. **Choose protocol wisely**: 开发/测试用 TCP，生产用 RDMA
7. **Monitor leases**: 对象有 TTL，必要时续租
8. **Handle errors**: 检查返回值并处理失败

---

## 文档链接

- Full API Reference: https://kvcache-ai.github.io/Mooncake/
- Quick Start: docs/source/getting_started/quick-start.md
- Mooncake Store: docs/source/python-api-reference/mooncake-store.md
- Transfer Engine: docs/source/python-api-reference/transfer-engine.md
- Transfer Engine direct usage: docs/source/design/transfer-engine/index.md#using-transfer-engine-in-your-projects
- EP Backend: docs/source/python-api-reference/ep-backend.md
