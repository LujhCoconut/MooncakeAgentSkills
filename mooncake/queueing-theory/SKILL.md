# Mooncake 排队论建模与分析

对 Mooncake 中所有排队点进行系统建模，用排队论估算延迟、识别瓶颈。

> **理论基础**：排队论是分析系统延迟和吞吐的核心数学工具。Mooncake 作为一个分布式系统，天然存在多个排队点——从 RDMA 硬件队列到软件调度队列。理解每个队列的行为模式，才能定位真正的瓶颈。

---

## 一、Mooncake 系统中的所有排队点

```
                         入向请求
                            │
    ┌───────────────────────┼───────────────────────┐
    │                       │                       │
    ▼                       ▼                       ▼
┌────────┐           ┌────────────┐          ┌──────────┐
│Conductor│          │   Store    │          │ Transfer │
│        │           │            │          │  Engine  │
│ Q11-13 │           │  Q6-Q10    │          │  Q1-Q5   │
└────────┘           └────────────┘          └──────────┘
```

## 二、排队论记号与公式速查

| 模型 | 含义 | 适用场景 | 关键公式 |
|------|------|---------|---------|
| **M/M/1** | Poisson 到达, 指数服务, 1 服务器 | 随机请求、简单队列 | W = 1/(μ-λ), L = ρ/(1-ρ) |
| **M/M/c** | Poisson 到达, 指数服务, c 服务器 | 多线程/多连接 | W = (C(c,ρ)/c)/(cμ-λ) + 1/μ |
| **M/D/1** | Poisson 到达, 确定服务, 1 服务器 | 固定处理时间 | W = (2-ρ)/(2μ(1-ρ)) |
| **M/G/1** | Poisson 到达, 一般服务, 1 服务器 | 服务时间有方差 | W = (λ·E[S²])/(2(1-ρ)) |
| **D/M/1** | 确定到达, 指数服务, 1 服务器 | 定时批量到达 | 复杂，查表 |

其中: λ = 到达率, μ = 服务率, ρ = λ/μ = 利用率, W = 平均等待时间, L = 平均队列长度

**Pollaczek-Khinchine (P-K) 公式 (M/G/1)**:
```
E[W] = (λ · E[S²]) / (2(1-ρ))
其中 E[S²] = Var[S] + E[S]²
```

**Erlang C 公式 (M/M/c)**:
```
C(c,ρ) = 排队概率 = P(wait > 0) = ( (cρ)^c / c! ) / ( (1-ρ) · Σ_{k=0}^{c-1} (cρ)^k/k! + (cρ)^c/c! )
```

---

## 三、Transfer Engine 排队点

### Q1: RDMA Send Queue (per QP)

**建模**: M/M/1（或 M/D/1，如果 WQE 处理时间确定）

**参数**:
- λ = 应用层 submitTransfer 的速率 (transfers/s)
- μ = NIC 处理 WQE 的速率（取决于消息大小和 NIC 速度）
  - 小消息 (4KB, inline): μ ≈ 1/(1µs) = 1,000,000/s
  - 中消息 (64KB): μ ≈ 1/(5µs) = 200,000/s
  - 大消息 (1MB): μ ≈ 1/(50µs) = 20,000/s
- c = QP 数量（如果多 QP 并发）

**延迟估算**:
```
RDMA 传输端到端延迟 = W_wait + W_service + W_wire

W_service (服务时间) = 消息大小 / 有效带宽
W_wire (线上延迟) ≈ RTT/2 ≈ 1-5µs (取决于距离和交换机跳数)
W_wait (排队等待) = ρ/(μ(1-ρ))   [M/M/1]


示例: 大消息传输 (1MB), λ=10,000/s, μ=20,000/s, ρ=0.5
  W_wait = 0.5/(20000×(1-0.5)) = 50µs
  W_service = 50µs
  W_wire = 3µs
  → 总延迟 ≈ 103µs

示例: 高负载 (ρ=0.9)
  W_wait = 0.9/(20000×(1-0.9)) = 450µs  ← 排队主导!
```

**瓶颈指标**: ρ > 0.7 → 排队延迟急剧上升

**优化杠杆**:
- 增加 QP 数量 (c↑): M/M/c → 降低排队
- 批量提交 WQE: 减少 λ（等效降低到达率）
- 用 NVLink 替代 RDMA: μ↑（更高服务率）

### Q2: RDMA Completion Queue (CQ Poll)

**建模**: 批量轮询 (Batch Polling)，等效 M/D/1

**参数**:
- 轮询间隔: T_poll ≈ MC_CQ_POLL_BATCH / μ_cq
- 每次 poll 处理: batch_size 个 completion
- 等效等待: W_cq ≈ T_poll/2 + batch内处理时间

**延迟估算**:
```
如果 MC_CQ_POLL_BATCH = 8, μ_cq = 10M/s
  T_poll = 8/10M = 0.8µs

总线式 polling (busy-poll): T_poll ≈ 0µs, 但 CPU 100%
事件式 (event-driven): T_poll ≈ 10-50µs (中断延迟)

推荐: 自适应 polling (低负载时 event, 高负载时 busy-poll)
```

### Q3: TENT Slice Scheduler Queue

**建模**: M/G/1（切分大小不同 → 服务时间方差大）

**参数**:
- λ = 传输请求到达率
- μ = 1/E[S], 其中 S 是切片的传输时间
- E[S²] = 服务时间的二阶矩（取决于切片大小的分布）

**延迟估算 (P-K 公式)**:
```
E[W] = (λ · E[S²]) / (2(1-ρ))

固定切片大小 (M/D/1):
  Var[S] = 0, E[S²] = E[S]²
  E[W] = (λ·E[S]²)/(2(1-ρ)) = (ρ·E[S])/(2(1-ρ))

可变切片大小 (M/G/1, Var[S] > 0):
  E[W] = (λ·(Var[S] + E[S]²))/(2(1-ρ))
  = E[W]_fixed · (1 + Var[S]/E[S]²)  ← 方差惩罚因子


示例: ρ=0.6, E[S]=100µs, Var[S]=0 (固定切片)
  E[W] = (0.6×100)/(2×0.4) = 75µs

如果切片大小可变 (Var[S] = E[S]², 即 Cv=1):
  方差惩罚 = 1+1 = 2×
  E[W] = 75 × 2 = 150µs  ← 方差导致等待时间翻倍!
```

**结论**: 使用固定或低方差的切片大小可以显著降低排队延迟。

### Q4: Memory Pin Request Queue

**建模**: M/M/1（首次 pin）或 M/D/1（cached pin）

**参数**:
- λ = 新内存区域的注册请求率
- μ_first ≈ 1/(1-50µs) = 20K-1M/s (取决于区域大小)
- μ_cached ≈ 1/(0.1µs) = 10M/s (pin cache 命中)

**关键**: Pin cache 的命中率是决定因素。
```
E[W] = h × 0 + (1-h) × E[W_first]
其中 h = pin cache 命中率

如果 h=0.9, E[W_first]=50µs, ρ=0.5:
  E[W] = 0.1 × (0.5/(20K×0.5)) = 0.1 × 50µs = 5µs
如果 h=0 (全部 miss):
  E[W] = 50µs  ← 10× 差距
```

### Q5: Batch Assembly Queue

**建模**: D/M/1 或 G/G/1（取决于 batch 策略）

**参数**:
- Batch 策略: 等够 batch_size 个请求，或等够 timeout
- 平均 batch assembly 延迟: T_assemble = min(batch_size/λ, timeout)

```
如果 batch_size=16, λ=100,000/s, timeout=100µs:
  填满 16 个需要 16/100K = 160µs > 100µs timeout
  → T_assemble = 100µs (timeout 触发)
  
如果 λ=1,000,000/s:
  填满 16 个需要 16/1M = 16µs < 100µs
  → T_assemble = 16µs (batch full 触发)
```

---

## 四、Store 排队点

### Q6: Master Request Queue

**建模**: M/M/c（c = Master 工作线程数）

**参数**:
- λ = 所有 client 的 Master 请求总和 (lookup + allocate + renew)
- μ = Master 单线程处理速率 ≈ 1/(10-100µs) = 10K-100K/s
- c = Master 线程数（默认需验证，假设 4-8）

**延迟估算**:
```
利用 Erlang C 公式:
  P(wait > 0) = C(c, ρ)

如果 λ=50,000/s, μ=20,000/s, c=4:
  ρ = 50000/(4×20000) = 0.625
  P(wait > 0) ≈ 10-15%
  E[W] ≈ 10-30µs (短)

如果 λ=70,000/s, μ=20,000/s, c=4:
  ρ = 0.875
  P(wait > 0) ≈ 40-50%
  E[W] ≈ 100-200µs  ← Master 成为瓶颈!
```

**瓶颈指标**: Master QPS 接近 c × μ，或 P(wait > 0) > 30%

**优化杠杆**:
- 增加 Master 线程 (c↑)
- Client-side metadata cache (λ↓)
- 读写分离 (lookup 和 allocate 用不同 path)

### Q7: Segment Allocation Queue

**建模**: M/M/1（默认）或 M/M/c（如果有并行分配能力）

**特定特征**:
- 分配请求 = put() 操作的一部分 → 延迟直接影响推理
- 如果预分配 pool > 0: W → 0 (从 pool 直接取)
- 如果 pool empty: W = M/M/1

**延迟估算**:
```
prealloc hit: W ≈ 0
prealloc miss: W = M/M/1 排队时间

如果 prealloc_pool_size=10, λ=1000/s, μ=500/s, ρ=2.0 (不稳定!)
  → 需要 pool 来吸收 burst
  → pool hit rate = 1 - P(empty pool)
     P(empty) ≈ exp(-10×(1-ρ)) ≈ exp(-10×(-1)) = exp(10) ≈ 0? 
     → 不对，ρ > 1 时 pool 会被耗尽 → prealloc miss 时系统不稳定

建议: prealloc_pool_size > λ_burst / μ
```

### Q8: Eviction Processing Queue

**建模**: M/D/1（驱逐检查是定时触发的）

**参数**:
- 驱逐检查间隔: T_check = MC_MASTER_EVICTION_CHECK_INTERVAL (默认 5s)
- 每次驱逐耗时: T_evict = 被驱逐对象数 × 单对象驱逐时间

**延迟不敏感**: 驱逐不在关键路径上（不阻塞读写），但驱逐不及时 → 空间不足 → put 失败。

```
驱逐速率需要 ≥ 对象淘汰速率:
  μ_evict ≥ λ_new × (1 - hit_rate)

如果 λ_new = 1000 objects/s, hit_rate = 0.7:
  需要驱逐 300 objects/s
  单对象驱逐 ≈ 10µs → μ_evict ≈ 100,000/s → 充裕

如果驱逐检查间隔过长 → 驱逐不及时 → G2/G3 满:
  需要的驱逐间隔 < capacity / (λ_new × (1-hit_rate) × avg_obj_size)
```

### Q9: Metadata Backend Queue (ETCD/Redis)

**建模**: M/M/c（ETCD 集群, c=节点数）

**参数**:
- λ = Master 发往后端的读写请求率
- μ_etcd: ETCD 单节点 ≈ 10,000-50,000 ops/s
- μ_redis: Redis 单节点 ≈ 50,000-100,000 ops/s

**延迟估算**:
```
ETCD (强一致, Raft):
  每次写需要 Raft 共识 → 额外 1-2 RTT
  W_etcd_write = W_queue + RTT_raft + W_disk
  RTT_raft ≈ 1-5ms (取决于节点间网络)

Redis (异步复制):
  W_redis = W_queue + W_memory
  几乎纯内存操作 → < 1ms

如果 metadata 以读为主 → ETCD 和 Redis 差异不大
如果 metadata 以写为主 → ETCD 有 Raft 开销, Redis 更快但弱一致
```

### Q10: Client Read/Write Queue

**建模**: M/M/1（单连接）或 M/M/c（连接池, c=pool size）

**延迟估算**:
```
E[W_client] = E[W_master_lookup] + E[W_transfer] + E[W_metadata_update]

每一步都是独立的排队点 → 端到端延迟 = 各步延迟之和（串联队列）

get() 路径:
  W_total = W_master_lookup(M/M/c) + W_transfer(M/M/1) = 预估值

put() 路径:
  W_total = W_master_lookup + W_segment_alloc + W_transfer + W_metadata_update
```

---

## 五、Conductor 排队点

### Q11: ZMQ Event Queue

**建模**: M/M/1

**参数**:
- λ = Store Put/Evict 事件的速率（高频）
- μ = ZMQ 消费速率 ≈ 1/(1-5µs) = 200K-1M events/s

**瓶颈条件**: 如果 Evict 事件风暴 → λ 激增 → ZMQ 积压 → 索引更新滞后

```
积压深度: L = ρ/(1-ρ)
如果 ρ=0.9: L = 9 个事件在排队
排队延迟: W = ρ/(μ(1-ρ)) = 0.9/(200K×0.1) = 45µs ← 可接受

如果 ρ=0.99: L = 99, W = 0.99/(200K×0.01) = 495µs ← 开始有问题
```

### Q12: Index Update Queue

**建模**: M/D/1（更新操作时间较固定）

**参数**:
- λ = 需要更新索引的事件率 ≈ 来自 Q11 的吞吐
- μ ≈ 1/(1µs) = 1M updates/s (hash table insert/delete)

**延迟估算**:
```
E[W] = (2-ρ)/(2μ(1-ρ))  [M/D/1]
ρ=0.5: E[W] = 1.5/(2M×0.5) = 1.5µs
ρ=0.9: E[W] = 1.1/(2M×0.1) = 5.5µs
```

### Q13: HTTP Query Queue

**建模**: M/M/c（c = Conductor HTTP 线程数）

**参数**:
- λ = 推理框架的 `/query` 请求率 ≈ 每个新 request 1次
- μ = Conductor 查询延迟 ≈ 1/(10-50µs) = 20K-100K/s
- c = HTTP 线程数

**端到端影响**:
```
Conductor 查询在 Prefill 的关键路径上:
  Prefill 延迟 += W_conductor_query + W_store_get(如果命中)

如果 Conductor 查询延迟增加 100µs:
  每个请求的 Prefill 延迟 +100µs
  大规模 batch (128 requests) 下:
    如果串行查询 → +12.8ms  ← 不可接受
    如果批量查询 → +100µs (并行) ← 可接受
```

---

## 六、端到端延迟建模

### Prefill 路径 (假设 PD 分离，KV cache 从 Store 获取)

```
端到端 Prefill 延迟 =
  W_conductor_query (Q13)                                [Conductor]
+ W_store_get_lookup (Q6)                                [Master]
+ W_store_get_transfer (Q10: W_transfer)                 [Transfer Engine]
+ W_transport_queue (Q1)                                 [RDMA/TCP]
+ Prefill 计算时间                                        [GPU]

如果不命中 (Cache Miss):
  额外: W_segment_alloc (Q7) + W_put_transfer (Q10) + W_metadata_write (Q9)
```

### Decode 路径 (每 token)

```
Decode 单 token 延迟 =
  GPU 计算时间
+ KV cache 读取 (如果在 local G1: ~0; 如果在 remote G2: +Q10)
+ 采样时间

每个 token 约 10-50ms → 排队延迟通常不是瓶颈
但如果 KV cache 在 remote → 每 token 都要 RDMA read → 严重瓶颈
```

### 串行队列的总延迟

```
串联系统的总平均等待时间 = Σ W_i (各环节等待时间之和)
串联系统的总利用率: ρ_system = max(ρ_i)  (最繁忙的环节决定吞吐)

瓶颈 = argmax_i(ρ_i)  → 利用率最高的环节 → 优化优先目标
```

### 识别瓶颈的方法

```
1. 计算每个排队点的 ρ_i = λ_i / (c_i · μ_i)
2. 计算每个排队点的 W_i
3. 找出 ρ_i 最大的环节 → 这就是瓶颈
4. 对比 W_i 在总延迟中的占比 → 最大的优化杠杆

常见瓶颈:
  - ρ_master > 0.8 → Master 瓶颈 (增加线程或客户端缓存)
  - ρ_rdma_qp > 0.9 → RDMA 瓶颈 (增加 QP 或升级带宽)
  - ρ_pin > 0.7 → Memory pin 瓶颈 (增加 pin cache)
  - ρ_eviction < required → 驱逐太慢 (降低 TTL 或增加检查频率)
```

---

## 七、优化决策树（基于排队论）

```
当前系统延迟超标
│
├─ 检查 ρ_master > 0.7?
│  └─ 是 → 优先优化 Master
│     ├─ 增加 Master 线程数 (c↑)
│     ├─ 启用 client-side metadata cache
│     └─ 考虑读写分离
│
├─ 检查 ρ_rdma > 0.8?
│  └─ 是 → 优先优化 RDMA 路径
│     ├─ 增加 QP 数量或使用 multi-rail
│     ├─ 升级网卡 (200G→400G)
│     ├─ 优化消息大小 (adaptive inline / write)
│     └─ 降低批量 assembly 延迟
│
├─ 检查 W_pin / W_total > 20%?
│  └─ 是 → 优先优化 Memory Pin
│     ├─ 增大 pin cache
│     ├─ 使用 huge pages
│     └─ Pre-pin 策略 (预测性 pin)
│
├─ 检查 TTL 驱逐不及时?
│  └─ 是 → 优先调驱逐
│     ├─ 降低 soft TTL
│     ├─ 减少驱逐检查间隔
│     └─ 增加 segment 容量
│
└─ 检查 Conductor stale index > 5%?
   └─ 是 → 优先优化 Conductor
      ├─ 增大 ZMQ buffer 或批量处理
      ├─ 索引 TTL 与 Store TTL 对齐
      └─ 增加 Conductor 实例数
```

---

## 八、与 /advanced-optimize 的衔接

当用户调用 `/advanced-optimize "对 Mooncake 进行排队论分析"` 时：

1. 阅读本 SKILL.md 理解所有排队点
2. 搜索 Mooncake 源码中每个排队点的实现（队列深度、线程数、batch 等实际参数）
3. 估算或测量各环节的 λ、μ、c
4. 计算 ρ_i 和 W_i
5. 识别瓶颈（ρ 最大、W 占比最高）
6. 检索 domain_knowledge_agent 中关于排队论、调度优化的论文
7. 按提案模板生成优化方案
