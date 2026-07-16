# Mooncake 排队论建模与分析

对 Mooncake 中所有排队点进行系统建模。**输入硬件配置和工作负载，输出具体延迟估算和瓶颈排名。**

## 调用方式

```
/advanced-optimize "对 Mooncake 进行排队论分析"
```

或附带配置信息：
```
/advanced-optimize "排队论分析: ConnectX-7 200Gbps, 8×H100, Samsung PM9A3 NVMe,
                   batch_size=64, avg_prompt=2000t, avg_decode=500t, QPS=50"
```

---

## 一、场景识别路由

首先解析用户输入，提取以下信息，缺失项使用典型默认值或主动询问：

### 1.1 提取配置信息

| 类别 | 参数 | 用户输入 | 默认值 | 用途 |
|------|------|---------|--------|------|
| **网卡** | NIC 型号 | ConnectX-7 / ConnectX-6 / EFA / 无RDMA | ConnectX-7 | Q1 μ 值 |
| **网卡** | 链路速度 | 200Gbps / 400Gbps / 100Gbps | 200Gbps | 带宽计算 |
| **RDMA** | QP 数量 | 1-8 | 4 | Q1 c 值 |
| **GPU** | 型号+数量 | H100×8 / A100×8 / H200×8 | H100×8 | G1 容量、计算速率 |
| **GPU** | NVLink 版本 | 4.0 / 3.0 / 无 | 4.0 | NVLink μ |
| **CPU** | NUMA 节点数 | 1 / 2 / 4 | 2 | NUMA 惩罚因子 |
| **DRAM** | G2 容量 | GB | 512 GB | G2 容量上限 |
| **SSD** | NVMe 型号 | PM9A3 / P5800X / 通用 | PM9A3 | Q_G3 μ 值 |
| **SSD** | NVMe 数量 | 1-8 | 4 | G3 总带宽 |
| **NVMe** | 裸盘 / 文件系统 / SPDK | 裸盘 | 裸盘 | I/O 路径延迟 |

### 1.2 提取工作负载信息

| 参数 | 含义 | 默认值 | 用途 |
|------|------|--------|------|
| `batch_size` | 推理批次大小 | 64 | λ 推导 |
| `avg_prompt_tokens` | 平均 prompt 长度 (tokens) | 2000 | KV cache 大小、λ_put |
| `avg_decode_tokens` | 平均生成长度 (tokens) | 500 | KV cache 存活时间 |
| `QPS` | 每秒推理请求数 | 50 | 全局 λ |
| `prefix_hit_rate` | 前缀缓存命中率 | 0.5 | λ_get vs λ_put |
| `model` | 模型名/大小 | Llama-70B | KV cache 每 token 大小 |
| `pd_disagg` | 是否 PD 分离 | true | 是否触发跨节点传输 |
| `replica_count` | 副本数 | 1 | 写入放大 |
| `store_ttl_hard` | Store 硬 TTL (s) | 5 | 驱逐参数 |
| `store_ttl_soft` | Store 软 TTL (s) | 1800 | 驱逐参数 |

---

## 二、硬件 → 参数映射表

### 网卡 → Q1 (RDMA Send Queue) μ 值

| NIC | 速度 | μ_small (4KB inline) | μ_mid (64KB) | μ_large (1MB) | RTT Wire |
|-----|------|---------------------|--------------|---------------|----------|
| ConnectX-7 | 400Gbps | 1,200,000/s | 250,000/s | 30,000/s | 1.5µs |
| ConnectX-7 | 200Gbps | 1,000,000/s | 200,000/s | 20,000/s | 2µs |
| ConnectX-6 Dx | 200Gbps | 900,000/s | 180,000/s | 18,000/s | 2.5µs |
| ConnectX-6 Dx | 100Gbps | 600,000/s | 120,000/s | 10,000/s | 3µs |
| ConnectX-5 | 100Gbps | 500,000/s | 100,000/s | 8,000/s | 3.5µs |
| AWS EFA | 100Gbps | 400,000/s | 80,000/s | 6,000/s | 8µs |
| TCP (无 RDMA) | 任意 | 50,000/s | 10,000/s | 500/s | 30µs |

### NVMe SSD → G3 I/O 参数

| NVMe 型号 | 4K 随机读 IOPS | 4K 随机写 IOPS | 顺序读 BW | 顺序写 BW | 延迟 (QD=1) |
|-----------|:------------:|:------------:|:---------:|:---------:|:----------:|
| Intel P5800X (Optane) | 1,500,000 | 1,300,000 | 7.4 GB/s | 6.5 GB/s | 5µs |
| Samsung PM9A3 | 750,000 | 160,000 | 3.5 GB/s | 1.75 GB/s | 70µs |
| Samsung PM1733 | 850,000 | 200,000 | 7.0 GB/s | 3.8 GB/s | 75µs |
| Kioxia CM6 | 800,000 | 180,000 | 6.5 GB/s | 3.5 GB/s | 80µs |
| 通用 datacenter NVMe | 500,000 | 100,000 | 3.0 GB/s | 1.5 GB/s | 80µs |

### GPU → KV cache 计算

| GPU | HBM 容量 | HBM 带宽 | NVLink BW | Prefill 计算速率 (Llama-70B) |
|-----|:------:|:-------:|:---------:|:--------------------------:|
| H100 (SXM) | 80 GB | 3.35 TB/s | 900 GB/s (4.0) | ~50K tokens/s |
| H200 | 141 GB | 4.8 TB/s | 900 GB/s (4.0) | ~60K tokens/s |
| A100-80GB | 80 GB | 2.0 TB/s | 600 GB/s (3.0) | ~30K tokens/s |
| A100-40GB | 40 GB | 1.5 TB/s | 600 GB/s (3.0) | ~30K tokens/s |

### 模型 → KV cache 大小

| 模型 | 层数 | hidden_dim | KV/token (FP16) | 2000t KV cache |
|------|:---:|:---------:|:---------------:|:--------------:|
| Llama-70B | 80 | 8192 | ~2.5 MB | ~5 GB |
| Llama-8B | 32 | 4096 | ~0.5 MB | ~1 GB |
| Qwen-72B | 80 | 8192 | ~2.5 MB | ~5 GB |
| DeepSeek-V2 | 60 | — | ~3.0 MB | ~6 GB |

---

## 三、工作负载 → λ 推导

根据用户提供的工作负载参数，自动推算出各排队点的到达率：

```
KV cache 每对象大小 = layers × 2 × hidden_dim × 2 (K+V) × 2 bytes (FP16) × prompt_tokens
                   ≈ avg_prompt_tokens × KV_per_token

λ_put (写请求) = QPS × (1 - prefix_hit_rate) × replica_count
  // 仅当 cache miss 时才写新 KV cache; 副本数会放大写入

λ_get (读请求) = QPS × prefix_hit_rate × avg_decode_tokens
  // 每次 cache hit 都需要 get; decode 每 token get 一次

λ_lookup (Master 查询) = λ_put + (QPS × prefix_hit_rate)
  // 每个 put 前查一次 + 每个可能 hit 的请求查一次

λ_renew (租约续期) = (QPS × (1-prefix_hit_rate)) × (avg_decode_tokens / TTL_hard)
  // 每个新建的 KV cache 对象在 decode 期间定期续期

λ_evict (驱逐) = λ_put (稳态)
  // 稳态下驱逐速率 ≈ 写入速率

λ_conductor_query = QPS
  // 每个推理请求查询一次 Conductor

λ_conductor_event = λ_put + λ_evict
  // Store 的每个 Put 和 Evict 都产生事件

传输大小分布:
  - 小消息 (<4KB): conductor query, metadata lookup, lease 续期
  - 中消息 (64KB-1MB): 单 token KV cache 读取 (get 单个 block)
  - 大消息 (>1MB): 完整 KV cache 写入 (put), Prefill→Decode 传输
```

**示例（默认参数）**:

```
QPS=50, batch_size=64, prefix_hit_rate=0.5, replica_count=1
avg_prompt_tokens=2000, avg_decode_tokens=500
模型: Llama-70B, KV_per_token ≈ 2.5 MB

→ KV cache 每对象 ≈ 2000 × 2.5MB = 5 GB
→ λ_put = 50 × 0.5 × 1 = 25 次/s
→ λ_get = 50 × 0.5 × 500 = 12,500 次/s
→ λ_lookup = 25 + 50×0.5 = 50 次/s
→ λ_renew = 25 × (500/5) = 2,500 次/s
→ λ_conductor_query = 50 次/s
→ λ_conductor_event = 25 + 25 = 50 events/s
```

---

## 四、逐排队点延迟估算引擎

> **用户提供实际参数时，代入精确计算；缺失参数时使用默认值并标注为估计值。**

### Q1: RDMA Send Queue (per QP) — `transfer-engine/transport/`

**模型**: M/M/c（c = QP 数量）

**参数来源**: NIC 型号 → μ 查表；工作负载 → λ 推导；QP=MC_QP_COUNT（默认4）

```
μ = 查表(NIC型号, 消息大小)
c = QP 数量
λ_qp = λ_transfer / c  (假设均分到各 QP)
ρ = λ_qp / μ

E[W] = C(c,ρ) / (cμ - λ) + 1/μ   [M/M/c Erlang C]
Total delay = E[W] + RTT/2

示例 (ConnectX-7 200Gbps, c=4, 大消息 5GB put):
  μ = 20,000/s, λ_put = 25/s, λ_per_qp = 6.25/s
  ρ = 6.25/20000 = 0.0003 → 几乎无排队
  E[W] ≈ 1/μ = 50µs, Total = 50 + 2 = 52µs

示例 (ConnectX-7 200Gbps, c=4, 中消息 get token, λ_get=12,500/s):
  λ_per_qp = 3,125/s, μ = 200,000/s
  ρ = 3125/200000 = 0.016
  E[W] ≈ 1/μ = 5µs, Total = 5 + 2 = 7µs

示例 (高负载, 仅 1 QP, λ_get=50,000/s):
  μ = 200,000/s, ρ = 0.25
  E[W] = 0.25/(200000×(1-0.25)) = 1.7µs + 5µs = 6.7µs  ← 仍不大
```

> **结论**: 对于 KV cache 场景，RDMA 发送队列通常不是瓶颈（μ >> λ）。除非消息极小+QPS极高。

---

### Q2: RDMA Completion Queue (CQ Poll) — `transfer-engine/`

**模型**: 批量轮询延迟

```
T_poll = MC_CQ_POLL_BATCH / μ_cq_ops

Busy-poll 模式 (默认):  CPU 100%, T_poll ≈ 0.3-0.8µs
Event-driven 模式:      CPU 低,  T_poll ≈ 10-50µs (中断)
Adaptive (推荐):         低负载 event, 高负载 busy-poll

E[W_cq] ≈ T_poll/2 (平均等待半次轮询周期)

示例 (Busy-poll, MC_CQ_POLL_BATCH=8, μ=10M/s):
  T_poll = 8/10M = 0.8µs
  E[W_cq] = 0.4µs

示例 (Event-driven, 中断延迟 ~20µs):
  E[W_cq] = 10µs   ← 但每个消息多 10µs!
```

> **注意**: `MC_CQ_POLL_BATCH` 在 MC_RDMA 相关参数中。如果使用 busy-poll，需检查 CPU 使用率是否可接受。

---

### Q3: TENT Slice Scheduler Queue — `tent/runtime/`

**模型**: M/G/1（需要切片大小的分布信息）

```
仅在大消息传输时触发（> 默认阈值）
切片大小: 如果用户未提供 → 假设 1MB (默认)
方差: 如果固定切片大小 → M/D/1; 如果自适应 → M/G/1

E[W] = (λ · (Var[S] + E[S]²)) / (2(1-ρ))

固定切片 (M/D/1):  E[W] = (ρ·E[S])/(2(1-ρ))
可变切片 (M/G/1):  E[W] = E[W]_fixed · (1 + Cv²)

示例 (5GB KV cache, 1MB 固定切片, μ=20,000/s, ρ=0.001):
  E[W] ≈ 0  ← 吞吐远大于需求

示例 (自适应切片, Cv=0.8, ρ=0.6):
  方差惩罚 = 1+0.64 = 1.64×
  固定时 E[W]=75µs → 可变时 E[W]=123µs
```

> **建议**: 如果使用 TENT 切片调度，固定切片大小的延迟更可控。

---

### Q4: Memory Pin Request Queue — `transfer-engine/memory/`

**模型**: M/M/1（首次 pin）或 ≈ 0（cached）

```
关键参数: pin cache 命中率 h, pin cache 大小 MC_MEMORY_PIN_CACHE_SIZE (默认 64)

μ_first = 查表(内存大小)  // 1MB ≈ 26µs, 100MB ≈ 2.5ms
μ_cached ≈ 10M/s (几乎无开销)

E[W] = h × 0 + (1-h) × E[W_first]
E[W_first] = M/M/1: ρ_pin / (μ_first × (1-ρ_pin))

示例 (5GB KV cache, 首次 pin=125ms, h=0.95, λ_pin=25/s):
  E[W_first] = 125ms (单个)
  E[W] = 0.05 × 125ms = 6.25ms ← 仍然不小!

示例 (h=0.99 优化后):
  E[W] = 0.01 × 125ms = 1.25ms

示例 (使用 2MB huge pages, pin 开销降低 80%):
  μ_first = 25ms (替代 125ms)
  E[W] = 0.05 × 25ms = 1.25ms  (即使用原来较低的 h)
```

> **关键优化**: Pin cache 命中率从 95%→99%，或使用 huge pages，能减少几毫秒的 put 延迟。

---

### Q5: Batch Assembly Queue — `transfer-engine/`

**模型**: 批量组装延迟

```
T_assemble = min(
    batch_size / λ_batch_arrival,  // 攒满一批的时间
    MC_BATCH_TIMEOUT               // 超时触发 (如 100µs)
)

实际值取决于 MC_BATCH_SIZE (默认 16) 和 λ。

如果 λ_batch_arrival = λ_get + λ_put = 12,525/s:
  T_assemble = min(16/12525=1.28ms, 100µs) = 100µs

如果 λ_batch_arrival = 1,000,000/s:
  T_assemble = min(16/1M=16µs, 100µs) = 16µs
```

---

### Q6: Master Request Queue — `store/master/`

**模型**: M/M/c（c = 工作线程数）

```
λ_master = λ_lookup + λ_allocate + λ_renew + λ_evict_notify

λ_lookup  = 50/s  (见 §三)
λ_allocate = 25/s
λ_renew   = 2,500/s
λ_evict   = 25/s
→ λ_total ≈ 2,600/s

μ = 查源码确认 (假设 20,000/s 单线程处理速率)
c = MC_MASTER 线程数 (默认假设 4-8)

ρ = 2600/(8×20000) = 0.016 → 几乎无排队
E[W] ≈ 1/μ ≈ 50µs

高负载场景 (QPS=500, λ_total≈26,000/s):
ρ = 26000/(8×20000) = 0.1625
E[W] ≈ 70µs ← 仍很低

极端场景 (QPS=5000, λ_total≈260,000/s):
ρ = 260000/(8×20000) = 1.625 > 1 → 系统不稳定! 必须加线程或缓存!
```

---

### Q7: Segment Allocation Queue — `store/master/`

**模型**: M/M/1 + prealloc pool

```
λ_alloc = λ_put = 25/s (默认)
μ_alloc = 1/(segment分配时间) ≈ 查源码 (假设 1-5µs = 200K-1M/s)

Pool 模式:
  hit: W = 0
  miss: W = M/M/1

预分配 pool size = MC_MASTER_SEGMENT_PREALLOC (默认 0, 即无预分配)

如果 pool_size=0: W = 1/200K ≈ 5µs (很低)
如果 pool_size>0 且命中: W = 0 ← 消除分配延迟
```

---

### Q8: Eviction Processing Queue — `store/master/`

**模型**: M/D/1（定时触发，不在关键路径）

```
驱逐检查间隔: T_check = MC_MASTER_EVICTION_CHECK_INTERVAL (默认 5s)

驱逐速率需求: μ_evict ≥ λ_put = 25/s (稳态)

单个驱逐操作耗时 ≈ 10µs → μ_evict ≈ 100,000/s >> 25/s

驱逐不及时的风险检查:
  G2 填满时间 = capacity_G2 / (λ_put × avg_obj_size × (1-hit_rate))
  = 512GB / (25/s × 5GB × 0.5) = 512/(62.5) ≈ 8.2s

  驱逐检查间隔 5s < 8.2s → 来得及
  但如果 G2 仅 128GB → 128/(62.5) ≈ 2s < 5s → 驱逐来不及!
```

> **关键**: 驱逐检查间隔必须小于 G2/G3 填满时间。

---

### Q9: Metadata Backend Queue (ETCD/Redis) — `store/master/`

**模型**: M/M/c

```
λ_backend = λ_master 中对 metadata 的读写操作

ETCD (c=3 节点, Raft 共识):
  μ = 10,000-50,000 writes/s per node
  写延迟 = W_queue + RTT_raft(1-5ms) + W_disk(1-5ms)
  W_etcd_write ≈ 3-10ms  (Raft 主导)

Redis (c=1 或 c=3 sentinel):
  μ = 50,000-100,000 ops/s per node
  W_redis ≈ 0.1-1ms  (纯内存)

对于 λ_backend ≈ 2,600/s (默认):
  ETCD: 可行, 但每次写 ~5ms ← 在 put 关键路径上!
  Redis: 每次写 ~0.5ms ← 低很多

建议: 如果有高频 metadata 写操作，Redis 明显更快；
      如果一致性要求高（>95%），ETCD 更合适。
```

---

### Q10: Client Read/Write Queue — `store/client/`

**模型**: 串联队列（M/M/c 连接池 + M/M/1 传输）

```
put() 端到端延迟:
  W_total = W_master_lookup(Q6) + W_segment_alloc(Q7)
          + W_memory_pin(Q4) + W_rdma_write(Q1) + W_metadata_write(Q9)

get() 端到端延迟:
  W_total = W_master_lookup(Q6) + W_rdma_read(Q1)

示例 (默认参数, put 5GB KV cache):
  Q6:  50µs  (Master lookup)
  Q7:   5µs  (segment alloc)
  Q4:  6.25ms (memory pin, h=0.95, 5GB)
  Q1:  52µs  (RDMA write)
  Q9:  5ms   (ETCD metadata write)
  ─────────────────
  ≈ 11.4ms (pin 和 ETCD 主导)

示例 (优化后: h=0.99, Redis, huge pages):
  Q6:  50µs
  Q7:   0µs  (prealloc pool)
  Q4:  0.3ms (h=0.99 + huge pages)
  Q1:  52µs
  Q9:  0.5ms (Redis)
  ─────────────────
  ≈ 0.9ms  ← 12× 改进!
```

---

### Q11: ZMQ Event Queue (Conductor ← Store)

**模型**: M/M/1

```
λ_event = λ_put + λ_evict = 50 events/s (默认)
μ_zmq ≈ 200,000 events/s

ρ = 50/200K = 0.00025 → 无影响

高负载 (QPS=5000): λ_event = 5,000 events/s
ρ = 5000/200K = 0.025 → 仍然很低
```

> ZMQ 队列在绝大多数场景下不是瓶颈。

---

### Q12: Index Update Queue (Conductor — Hash Table)

**模型**: M/D/1

```
λ_update = λ_event = 50/s (默认)
μ_update ≈ 1M updates/s (哈希表 insert/delete)

E[W] = (2-ρ)/(2μ(1-ρ))
ρ=0.00005 → E[W] ≈ 1/μ = 1µs
```

---

### Q13: HTTP Query Queue (Conductor → 推理框架)

**模型**: M/M/c

```
λ_query = QPS = 50/s (默认)
μ_query ≈ 20,000-100,000/s (取决于索引大小)
c = HTTP 线程数 (假设 4)

ρ = 50/(4×50K) = 0.00025
E[W] ≈ 1/μ = 20µs

高负载 (QPS=5000, batch 查询):
  如果支持 batch query → λ_effective = 5000/batch_size
  如果每次查一个 → λ = 5000/s, ρ = 5000/(4×50K) = 0.025 → 仍低
```

---

### G3 额外排队点: NVMe I/O Queue — `store/storage-backend/`

**这是用户最常关心的 SSD 层排队点。**

**模型**: M/M/c（c = NVMe 硬件队列深度）

```
NVMe 硬件参数:
  QD (Queue Depth) = NVMe 控制器支持的队列深度 (通常 1024 per namespace)
  NC (Number of Commands) = 实际使用的并发命令数
  IOPS = 查表(NVMe 型号, I/O 大小)

λ_nvme = λ_put + λ_get_miss (所有命中 G3 的读写)
  // G3 仅在 G1+G2 miss 时才被访问
  λ_nvme = (λ_put + λ_get) × (1 - G1_hit - G2_hit)
  // 如果 G1_hit=0.7, G2_hit=0.2, G3_hit=0.1
  // λ_nvme = (25 + 12500) × 0.1 ≈ 1,253/s

μ_nvme = IOPS_per_q × min(NC, QD)

如果使用 PM9A3, 64KB 随机读 (KV cache block 读取):
  IOPS ≈ 50,000 (64KB random read)
  μ_nvme ≈ 50,000/s, c = 4 NVMe drives

ρ = 1253/(4×50000) = 0.006 → 几乎无排队

但如果 store KV cache 在 G3 上的 get 很多:
  λ_nvme = 100,000/s (高负载, 低 G1/G2 命中)
  ρ = 100K/(4×50K) = 0.5
  E[W] ≈ M/M/4: ~20-50µs  ← 可接受
```

**NVMe 写入路径延迟拆解**:

```
G3 write = 文件系统层 + NVMe 驱动 + 设备处理 + NAND 编程

使用裸盘 Direct I/O:
  W_g3_write ≈ 70µs (PM9A3, QD=1, 4KB)
               + 数据传输时间 (BW / 有效带宽)

使用文件系统 (如 ext4/xfs):
  W_g3_write ≈ 70µs + 日志写入 + 元数据更新 ≈ 100-200µs

使用 SPDK (用户态 NVMe 驱动):
  W_g3_write ≈ 5-20µs + 数据传输  ← 显著降低
```

> **建议**: G3 裸盘 + Direct I/O 是最优选择。文件系统不提供 KV 场景的价值（page cache 无用），反而增加延迟。SPDK 可进一步降低延迟但增加运维复杂度。

---

## 五、端到端延迟汇总与瓶颈排名

### 5.1 自动生成延迟瀑布图

根据用户提供的参数，生成每个排队点的延迟占比：

```
示例 (默认参数: QPS=50, H100×8, ConnectX-7 200Gbps, PM9A3, Llama-70B):

Put 路径延迟分解 (5GB KV cache):
  Q6 Master Lookup     ██ 50µs   (0.4%)
  Q7 Segment Alloc     █ 5µs     (0.0%)
  Q4 Memory Pin        ████████████████████████████████ 6250µs (54.5%)
  Q1 RDMA Write        █ 52µs    (0.5%)
  Q9 ETCD Write        ██████████████████████████████ 5000µs (43.6%)
  ────────────────────────────────────────
  合计: 11,357µs ≈ 11.4ms

  瓶颈 1: Q4 Memory Pin (54.5%) → 增加 pin cache, huge pages
  瓶颈 2: Q9 ETCD Write (43.6%) → 换 Redis 或异步写入

Get 路径延迟分解 (单 token, 64KB):
  Q6 Master Lookup     ██ 50µs  (87.7%)
  Q1 RDMA Read         █ 7µs    (12.3%)
  ────────────────────────────────────────
  合计: 57µs

  (非常低, 不是瓶颈)

Prefill Cache Hit 路径 (PD 分离, 5GB 传输):
  Q13 Conductor Query  █ 20µs   (0.3%)
  Q6 Master Lookup     ██ 50µs  (0.7%)
  Q1 RDMA Read(5GB)    ████████████████████████████████████ 5000µs (71.3%)
  Q4 Memory Pin(远端)  ████████████████ 1950µs (27.8%)
  ────────────────────────────────────────
  合计: ~7.0ms

  瓶颈 1: Q1 RDMA 传输 (71.3%) → 升级带宽或多路径
  瓶颈 2: Q4 Memory Pin (27.8%) → 远端 pin cache
```

### 5.2 瓶颈排名

按 ρ 值和延迟占比综合排名，自动生成：

```
排名 1: Q4 Memory Pin     — ρ_pin 忽略不计但延迟占比最高 (54.5%)
       优化: 增大 pin cache → 99%+, 使用 huge pages, pre-pin

排名 2: Q9 ETCD Write     — 延迟占比 43.6%, 每次 put 走 Raft 共识
       优化: 切换到 Redis(低延迟), 或异步写 metadata

排名 3: Q1 RDMA Transfer  — 大对象传输延迟主导 Prefill Cache Hit 路径
       优化: multi-rail(多QP), 升级 200G→400G, NVLink GPU直连

排名 4: Q6 Master Lookup  — 占比小但随 QPS 线性增长
       优化: client-side metadata cache (λ↓)

排名 5: Q8 Eviction       — 依赖 G2 容量, 驱逐间隔必须 < 填满时间
       优化: 确保 MC_MASTER_EVICTION_CHECK_INTERVAL < G2_fill_time
```

---

## 六、用户输入驱动的分析流程

当用户调用时，按以下步骤执行：

### Step 1: 提取配置

从用户输入中提取硬件和工作负载参数。对于未提供的参数：
- 优先使用表 1.1/1.2 中的默认值
- 标注 "未提供, 使用默认值 XXX"
- 如果参数对结论影响大（ρ 接近 1）, 提示用户 "建议提供精确值以得到更准确的估算"

### Step 2: 推导 λ

用 §三 中的公式，从 QPS/batch_size/hit_rate 推导出各排队点的到达率。

### Step 3: 查表得 μ

用 §二 中的硬件→参数映射，查出每个排队点的服务率。

### Step 4: 计算每个排队点

用 §四 中的公式，对 13+1 (G3) 个排队点逐一计算 ρ 和 W。

### Step 5: 生成汇总

1. **延迟瀑布图**（各排队点延迟在端到端路径中的占比）
2. **瓶颈排名**（ρ 最大 + 延迟占比最高的 3-5 个）
3. **优化建议**（针对每个瓶颈的具体措施、预期收益）
4. **灵敏度分析**（改变 1-2 个参数看延迟变化）

### Step 6: 检索领域知识

对排名靠前的瓶颈，检索 domain_knowledge_agent 中的相关论文：
- Memory pin 瓶颈 → `performance/system-tuning/KNOWLEDGE.md`
- ETCD 延迟 → `algorithms/distributed-consensus/KNOWLEDGE.md`
- RDMA 传输 → `network/os-networking/KNOWLEDGE.md`
- NVMe 延迟 → `performance/storage-filesystem/KNOWLEDGE.md`

### Step 7: 输出方案

按优化提案模板写入 `proposals/mooncake-queueing-analysis-<date>.md`。

---

## 七、场景速查

### 场景 A: PD 分离, Prefill→Decode KV cache 传输

```
输入: pd_disagg=true, ConnectX-7 200Gbps, H100×8, avg_prompt=2000t
瓶颈: Q1 RDMA + Q4 Pin (远端)
最优协议: NVLink (同节点) > RDMA (跨节点)
关键参数: QP 数量, multi-rail, MC_QP_COUNT
```

### 场景 B: 高 QPS, 频繁 Master 查询

```
输入: QPS=5000, batch_size=128
瓶颈: Q6 Master → Q6 ρ 可能接近 1
关键参数: MC_MASTER 线程数, client-side metadata cache
```

### 场景 C: KV cache 冷热明显, G3 频繁访问

```
输入: G2 容量不足, NVMe=PM9A3×4
瓶颈: Q_G3 + Q8 Eviction (驱逐不及时)
关键参数: MC_STORE_G2_CAPACITY, MC_MASTER_EVICTION_CHECK_INTERVAL,
          Direct I/O 是否启用, SPDK 收益
```

### 场景 D: 前缀缓存命中率低, Conductor 瓶颈

```
输入: prefix_hit_rate=0.1, 大量 Evict 事件
瓶颈: Q11 ZMQ + Q12 Index Update (事件积压)
关键参数: MC_CONDUCTOR_EVENT_BATCH_SIZE, MC_CONDUCTOR_INDEX_TTL
```
