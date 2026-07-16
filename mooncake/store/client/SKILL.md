# Mooncake Store — Client 数据路径优化

Client 是 Mooncake Store 的 **数据路径**——负责 put/get 操作的端到端执行。在分布式存储系统中，数据路径是决定端到端延迟和吞吐的关键。

> **理论根基**: 参见 `../architecture.md` §2.3 (控制与数据平面分离)、§3.2 (零拷贝路径)、§5 (三视角交叉点①)

## 分布式存储系统视角

### 数据路径在分布式存储中的角色

分布式存储系统的数据路径设计有一个经典模式：

```
控制平面: 客户端 → Master (请求元数据, 获取数据位置)
                                 │
数据平面: 客户端 ─────────────────┘→ 存储节点 (直接读写数据)
                                 (绕过 Master)
```

Mooncake 完全遵循这个模式：
- `put()` / `get()` 先通过 Master 获取 segment 信息
- 然后 Client 直接通过 Transfer Engine RDMA 读写存储节点的 DRAM/SSD
- Master 不参与数据搬运

### 数据路径的关键延迟构成

```
put() 端到端延迟 = Master查询 + 内存分配 + 数据传输 + 元数据更新

get() 端到端延迟 = Master查询 + 数据定位 + 数据传输
                  └── 可缓存 ──┘  └── 零拷贝 ──┘
```

**优化的主要杠杆**：
1. **减少 Master 查询**：Client-side metadata cache
2. **减少数据拷贝**：零拷贝路径 (RDMA read/write)
3. **减少往返 (RTT)**：请求 pipelining、batch

## 源代码地图

| 文件 | 说明 | 子组件 |
|------|------|--------|
| `mooncake-store/src/mooncake_client/real_client.cpp` | RealClient (RDMA + 内存直管) | client |
| `mooncake-store/src/mooncake_client/dummy_client.cpp` | DummyClient (RPC 代理) | client |
| `mooncake-store/src/mooncake_client/client_allocator.cpp` | 客户端侧内存分配器 | storage-backend |
| `mooncake-store/include/real_client.h` | RealClient 公共接口 | client |
| `mooncake-store/include/store_c.h` | C FFI | client |
| `mooncake-store/src/p2p_store.cpp` | P2P Store | client |
| `mooncake-store/src/transfer_task_scheduler.cpp` | 传输任务调度 | client |

## 优化维度

### 1. 读写路径优化 — 端到端延迟分析

- **代码入口**: `real_client.cpp:put()` / `get()` / `put_from()` / `get_into()`

**延迟拆解 (以 get 为例)**：
```
get(key) 的总延迟 = 
    lookup_master(key)                    # 查询 segment 位置
  + locate_segment_local(segment_id)      # 本地寻址
  + transfer_data(addr, length)           # RDMA read (零拷贝) 或内存拷贝
  + deserialize / validate (if any)       # 数据校验/反序列化
```

**每一步的优化空间**：
- `lookup_master`: Client-side cache + batch lookup → 减少 Master RPC
- `transfer_data`: 零拷贝路径验证 + 协议选择 → RDMA read vs. TCP
- `deserialize`: 如果传输的是 raw tensor bytes，是否可以消除反序列化？

**LLM 推理的读写模式特征**：
- `put()` 发生在 Prefill 完成后 → 延迟敏感（阻塞 Decode 开始）
- `get()` 发生在 Decode 期间 → **极度延迟敏感**（每次 token 生成都要读 KV cache）
- 读写比例：读 >> 写（每个 KV cache 写 1 次，读 N 次，N = 生成的 token 数）

**关键问题**:
- get() 路径是否真的零拷贝？需要端到端 trace 验证
- 小对象 (< 10KB, 短 prompt) vs. 大对象 (> 10MB, 长 prompt) 的读写路径是否区别对待？
- 是否有 request pipelining？（多个 get 复用同一 RDMA 连接）
- 读写的 NUMA 感知？（reader 所在的 CPU core 和 NIC 是否在同一 NUMA node）

**Domain Knowledge 映射**:
- `performance/system-tuning/KNOWLEDGE.md` — I/O 路径、NUMA、零拷贝
- `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU 数据搬运、GDR

### 2. 连接管理与多路复用

- **代码入口**: `real_client.cpp:setup()`

**分布式系统视角**：
- 每个 client 建立连接的开销 = RDMA QP 创建 + 内存注册 + metadata 交换
- 连接数 × 开销 → 集群规模扩大的瓶颈
- 解决方案：连接池 + 多路复用（单连接承载多个并发请求）

**多路复用的设计选择**：

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **Connection per Request** | 每次请求建立/销毁连接 | 简单 | 开销大 |
| **Connection Pool** | 复用已有连接 | 减少建连开销 | 需管理池大小 |
| **Multiplexing** | 单连接多请求 | 最少连接数 | 需要请求标识和响应匹配 |
| **Connection per Node** | 每节点一个连接 | 均衡 | 单连接带宽可能受限 |

**关键问题**:
- 当前连接建立的开销（RDMA QP 创建）？
- 连接池的最大连接数、空闲超时、健康检查？
- 是否支持单连接上的请求多路复用？
- 连接的心跳/keepalive 策略？

**Domain Knowledge 映射**:
- `operations/cloud-infrastructure/KNOWLEDGE.md` — 连接池、RPC 框架
- `performance/concurrency/KNOWLEDGE.md` — 并发连接模型

### 3. 批量操作调度

- **代码入口**: `real_client.cpp:batch_put_from()` / `batch_get_into()`

**分布式系统视角**：
- 批量操作的本质是「用吞吐换延迟」——牺牲单个请求的延迟来提升总体吞吐
- 类比 Group Commit (数据库)、Batched RPC (网络)

**批量调度的设计空间**：

```
┌─────────────────────────────────────────────┐
│ Batch Scheduler                              │
│                                              │
│ 输入: [req1, req2, ..., reqN]                │
│                                              │
│ 参数:                                        │
│  - 批量大小: 自适应 vs. 固定？                  │
│  - 等待时间: 多久凑一批？(batching delay)       │
│  - 调度顺序: FIFO vs. 排序 (大先/小先)？        │
│  - 失败处理: 部分成功 vs. 全回滚 vs. best-effort│
│                                              │
│ 输出: 分布式/并发执行                          │
└─────────────────────────────────────────────┘
```

**LLM 推理的批量特征**：
- 同一 Prefill batch 中的多个请求同时产生 KV cache → 自然形成的写入批量
- Decode batch 中的多个请求同时需要读 KV cache → 自然形成的读取批量
- **批量大小与推理 batch size 直接相关** → 调度器可感知上层 batch size

**关键问题**:
- 批量大小是否有上限？超过上限是否拆分？
- 批量内是否支持部分成功？（一个 key 失败不影响其他 key）
- 批量操作的排序？（按 key 排序可提升存储层局部性）
- 批量操作与单操作是否可以混用？

**Domain Knowledge 映射**:
- `performance/optimization-paradigms/KNOWLEDGE.md` — 批处理模式、流水线
- `algorithms/resource-scheduling/KNOWLEDGE.md` — 批量调度

### 4. RealClient vs. DummyClient — 两种数据路径

- **代码入口**: `real_client.cpp`, `dummy_client.cpp`

**分布式系统视角**：
- RealClient = Direct I/O 模式（客户端直接 RDMA 到存储节点）
- DummyClient = Proxy 模式（客户端通过 RPC 代理到 RealClient）
- 这类似于 Ceph 的 librbd (direct) vs. RGW (proxy)

**两种模式的取舍**：

| 维度 | RealClient | DummyClient |
|------|-----------|-------------|
| **延迟** | 低 (直接 RDMA) | 高 (RPC + 代理) |
| **资源需求** | 需要 RDMA 网卡 + 内存 pin | 仅需网络 |
| **适用场景** | GPU 节点 (Prefill/Decode) | CPU-only 节点、管理节点 |
| **单点风险** | 无 (直连) | 代理节点可能成为瓶颈 |

**关键问题**:
- DummyClient→RealClient 的代理是否有负载均衡？
- DummyClient 是否感知代理节点的 health？
- 两种模式是否可以动态切换？（RDMA 链路故障时 DummyClient fallback？）

**Domain Knowledge 映射**:
- `architecture/cloud-native/KNOWLEDGE.md` — 代理模式、sidecar

### 5. P2P Store — 去中心化数据共享

- **代码入口**: `p2p_store.cpp`

**分布式系统视角**：
- P2P Store 绕过了 Master 中心化的存取路径
- 适用于临时对象（checkpoint、中间结果）的共享
- 去中心化的设计挑战：发现机制、一致性、安全边界

**P2P 发现的设计空间**：

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **Master-mediated** | 通过 Master 查询 peer 位置 | 简单 | Master 压力 |
| **DHT (Distributed Hash Table)** | O(log N) 去中心化查找 | 扩展性好 | 实现复杂 |
| **Gossip** | 最终一致性的成员管理 | 容错高 | 收敛慢 |

**关键问题**:
- P2P 的 peer 发现是否经过 Master？（当前可能是 Master-mediated）
- P2P 传输的路径选择？（直连 vs. relay）
- P2P 共享的安全边界？（谁能访问我的 P2P 数据？）

**Domain Knowledge 映射**:
- `network/os-networking/KNOWLEDGE.md` — P2P 网络
- `architecture/distributed-systems/KNOWLEDGE.md` — 去中心化

### 6. Client 可观测性

- **关键指标**:
  - put/get 延迟 (p50/p99/p999)，分对象大小统计
  - put/get 操作的各部分延迟拆解（lookup vs. transfer vs. metadata）
  - 批量操作的吞吐 (ops/s) 和带宽 (GB/s)
  - 连接数、连接建立/断开速率、连接复用率
  - 错误率（按错误类型、按操作类型）
  - 零拷贝路径的覆盖率（多少比例的读写走零拷贝？）
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。


## ⚠️ 维护规则

**当本 SKILL.md 有实质性更新（新增优化维度、修改分析流程、更新路由规则、新增领域知识映射等），必须同步检查并更新 `README.md` 中对应的项目结构、组件覆盖表、使用示例或模块说明。** 保持 README 始终反映最新的 skill 能力边界。
