# Mooncake Conductor (KV Cache Indexer) — LLM Serving 视角

Conductor 是 Mooncake 中 **最贴近 LLM 推理语义** 的组件。它解决的核心问题是：在当前推理请求到达之前，判断它的 prompt 前缀是否已有缓存的 KV cache。从 LLM Serving 视角看，这是 **前缀感知路由层**——直接决定了 Prefill 延迟能否被加速。

> **理论根基**: 参见 `../architecture.md` §4.1 (KV cache 语义)、§4.2 (前缀缓存)、§4.4 (端到端延迟模型)

## LLM Serving 视角

### 前缀缓存在推理管线中的位置

```
用户请求到达
    │
    ▼
┌──────────────┐
│  Conductor    │ ← 查询: 此 prompt 的前缀是否命中?
│ (KV Indexer) │
└──────┬───────┘
       │
    ┌──┴──┐
    │     │
  HIT!  MISS!
    │     │
    ▼     ▼
  直接加载  完整 Prefill
  KV cache  (计算全部 attention)
  (O(1))    (O(N²))
    │     │
    └──┬──┘
       ▼
    Decode 开始
```

**量化收益**：
- 如果 system prompt 占 prompt 的 50%，且前缀缓存命中 → Prefill 节省 50%
- 如果 multi-turn 对话命中前几轮的 KV cache → Prefill 节省 80%+
- **Conductor 的价值 = 前缀命中的概率 × Prefill 计算量**

### KV Cache 索引的核心技术挑战

**挑战 1：查找效率**
- Token 序列长度：1K-128K tokens
- 如果对每个请求都做 O(N) 的前缀匹配 → 索引查询本身成为瓶颈
- 解决方案：哈希 → O(1) 查询

**挑战 2：哈希碰撞**
- XXH3-64 (64-bit hash) → 碰撞概率 ~10⁻¹⁰ (1 亿 keys 下约 0.1%)
- 但对于精确的 KV cache 匹配（token 完全一致才复用），任何碰撞都可能导致错误复用
- **这是正确性 vs. 效率的权衡**

**挑战 3：最长前缀匹配 vs. 精确匹配**
- 精确匹配：只有完全相同的 token 序列才算命中 → 简单但丢失部分命中机会
- 最长前缀匹配：匹配最长的公共前缀 → 更高的命中率但更复杂的查找

**挑战 4：索引与存储的一致性**
- Conductor 订阅 Store 的 Evict 事件来更新索引
- 事件到达 Conductor 之前有一个时间窗口 → 索引可能指向已不存在的 KV cache
- **需要处理过时索引（stale index）的不命中**

## 源代码地图

| 文件/资源 | 说明 |
|----------|------|
| `docs/design/conductor/conductor-architecture-design.md` | Conductor 架构设计 |
| `docs/design/conductor/indexer-api-design.md` | Indexer API 设计 |
| HTTP: `/register`, `/unregister`, `/query`, `/query_by_hash` | Indexer HTTP API |

### 关键组件（基于设计文档）
- `EventManager` — 订阅 KV cache 事件
- `ZMQClient` — 与 Store 的 ZMQ 通信
- `KVEventHandler` — 事件处理逻辑
- `PrefixCacheTable` — XXH3-64 滚动哈希索引表
- `ModelContext` — (tenant_id, model_name, lora_name, block_size, salt, instance_id)

## 优化维度

### 1. 索引结构与查找效率 — 最长前缀匹配

- **设计参考**: `indexer-api-design.md`, `PrefixCacheTable`

**索引结构对比**：

| 结构 | 查找复杂度 | 空间复杂度 | 最长前缀 | 增量更新 |
|------|----------|----------|---------|---------|
| **Hash Table** (当前) | O(1) avg | O(N) | ❌ | O(1) |
| **Trie / Radix Tree** | O(L) (L=length) | O(N×L) | ✅ | O(L) |
| **Adaptive Radix Tree (ART)** | O(L) | O(N×L) 较小 | ✅ | O(L) |
| **Minimal Perfect Hash** | O(1) | O(N) | ❌ | 需要重建 |
| **Hybrid: Hash + Radix** | O(1) 热点, O(L) 其他 | 中 | ✅ (Radix 部分) | 混合 |

**LLM 推理的前缀匹配特征**：
- 最常见的命中模式：system prompt 完全相同 → 精确匹配即可覆盖
- 次常见：多轮对话中前 N-1 轮完全相同 → 最长前缀匹配有价值
- 最罕见但最有价值：跨请求的部分前缀共享 → 最长前缀匹配但实现复杂

**实用建议**：
- **Tier 1 (现在可做)**: Hash Table with Coarse Hash Chain (解决 XXH3 碰撞问题)
- **Tier 2 (中期)**: Add Radix Tree for Longest Prefix Match (多轮对话场景)
- **Tier 3 (远期)**: Semantic-level prefix matching (bypass tokenization, 基于 embedding)

**关键问题**:
- XXH3-64 的碰撞处理？（Coarse Hash Chain: 相同 hash 的存链表，确认 exact match）
- 查询时是否支持最长前缀匹配？（/query 的 API 设计）
- 索引表的内存占用？（N 个 prompt 前缀 × 每个前缀的哈希链）
- 空间是否可以压缩？（token 序列去重？delta encoding？）

**Domain Knowledge 映射**:
- `algorithms/resource-scheduling/KNOWLEDGE.md` — 哈希、索引结构
- `algorithms/graph-processing/KNOWLEDGE.md` — Radix Tree, ART
- `performance/gpu-ai-performance/KNOWLEDGE.md` — KV cache, prefix caching

### 2. 缓存命中率的提升 — 从精确到智能

- **设计参考**: `PrefixCacheTable` 的匹配逻辑

**命中的层次模型**：

```
┌─────────────────────────────────┐
│ Level 1: 精确匹配 (Exact Match)   │ ← 当前实现
│   所有 token 完全相同              │   命中率: ~30-60%
├─────────────────────────────────┤
│ Level 2: 前缀匹配 (Prefix Match)  │ ← 待实现
│   前 N 个 token 相同              │   命中率: +10-20%
├─────────────────────────────────┤
│ Level 3: 子序列匹配 (Subsequence) │ ← 远期
│   允许中间插入/删除 token          │   命中率: +5-10%
├─────────────────────────────────┤
│ Level 4: 语义匹配 (Semantic)      │ ← 研究
│   system prompt "你是..." vs.     │   命中率: +5-15%
│   "你是一个..." → 语义相同         │   但实现极复杂
└─────────────────────────────────┘
```

**每个 Level 的代价**：
- Level 1: O(1) 哈希查找，无额外开销
- Level 2: O(L) 前缀树查找，需要额外的树结构
- Level 3: 编辑距离/最长公共子序列 (LCS) → 计算开销可能超过 Prefill 节省
- Level 4: 需要 embedding 模型 → 引入新的延迟

**实用建议**：先做到 Level 2（前缀匹配），因为多轮对话场景下命中率提升显著且实现代价可控。

**关键问题**:
- 当前仅支持精确匹配还是已支持前缀匹配？
- 缓存命中的统计：命中率、命中长度分布、miss 的原因分布？
- Conductor 是否感知 token 的 "语义位置"？（system prompt vs. user message vs. tool call）

**Domain Knowledge 映射**:
- `performance/gpu-ai-performance/KNOWLEDGE.md` — LLM inference optimization
- `algorithms/resource-scheduling/KNOWLEDGE.md` — matching algorithms

### 3. 事件处理管线的实时性

- **设计参考**: `EventManager`, `KVEventHandler`

**事件处理延迟的影响**：

```
Timeline:
  t0: KV cache 在 Store 中被驱逐 (Evict)
  t1: Store 发送 Evict 事件到 Conductor
  t2: Conductor 收到并处理事件
  t3: Conductor 更新索引，删除对应的条目

t3 - t0 之间的窗口内，Conductor 返回 "命中"
  → 框架去 Store get(key)
  → 返回 "not found" (已被驱逐)
  → Prefill 延迟增加 = Conductor 查询时间 + Store 查询时间 + Prefill 计算时间
```

**事件处理的吞吐要求**：
- 高负载下 Store 可能产生每秒数千个 Put/Evict 事件
- Conductor 需要能够跟上这个速率 → 否则事件积压 → 索引越来越不准

**关键问题**:
- 事件处理的延迟 (p50/p99)？积压时的最大延迟？
- 事件处理的并发模型？（单线程？多线程？批处理？）
- 事件丢失的检测和恢复？
- 是否可以跳过一些事件（如高频的 TTL 驱逐事件因为本身就接近过期）？

**Domain Knowledge 映射**:
- `performance/optimization-paradigms/KNOWLEDGE.md` — 事件驱动架构
- `operations/monitoring-observability/KNOWLEDGE.md` — 流水线监控

### 4. 与 Store 的一致性 — Stale Index 处理

**Stale Index 的来源**：
1. Evict 事件延迟到达（如 §3 所述）
2. 节点故障导致 KV cache 丢失但 Conductor 未收到事件
3. Lease 过期但 Evict 事件丢失

**不一致窗口的处理策略**：

| 策略 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **Accept & Retry** | 返回 hit，框架 get 失败后再 Prefill | 简单 | 浪费了一次失败的 get |
| **TTL on Index** | 给索引条目设 TTL，自动过期 | 有界的不一致窗口 | 可能过早无效化有效索引 |
| **Probe before Reply** | Conductor 先确认 Store 有数据再返回 | 始终准确 | 增加 Conductor 延迟 |
| **Negative Cache** | 记录最近失败的 key，Conductor 查询时过滤 | 学习式修复 | 增加复杂度 |

**关键问题**:
- 当前如何处理 stale index？
- Evict 事件丢失的检测？
- 是否可以给索引条目加 TTL（软过期策略）？

**Domain Knowledge 映射**:
- `architecture/distributed-systems/KNOWLEDGE.md` — 缓存一致性
- `algorithms/distributed-consensus/KNOWLEDGE.md` — 最终一致性

### 5. 多租户隔离与共享

- **设计参考**: `ModelContext` = (tenant_id, model_name, lora_name, block_size, salt, instance_id)

**多租户索引的设计选择**：

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **完全隔离** (当前) | 每个 tenant 独立索引 | 安全、无干扰 | 相同 system prompt 无法跨租户共享 |
| **模型级共享** | 同 model 的 tenant 共享索引 | 提升命中率 | 需要访问控制 |
| **可控共享** | tenant 可 opt-in 共享 | 灵活 | 配置复杂 |

**跨租户共享的 LLM 推理价值**：
- 同一 SaaS 平台的多个客户使用相同的 system prompt → 共享前缀缓存 → Prefill 时间节省
- 但需要确认：共享是否有安全/合规风险？（KV cache 中是否包含敏感信息？）

**关键问题**:
- 是否需要跨租户前缀共享？
- 如果需要，安全边界如何保证？
- ModelContext 的 salt 字段的设计用途？

**Domain Knowledge 映射**:
- `security/os-security/KNOWLEDGE.md` — 隔离、安全边界
- `architecture/cloud-native/KNOWLEDGE.md` — 多租户架构

### 6. Conductor 可观测性

- **关键指标**:
  - 缓存命中率（按 ModelContext 分组、按 token 位置分组）
  - 缓存命中的前缀长度分布
  - 查询延迟 (p50/p99/p999)
  - 索引大小 (条目数、内存占用)
  - 事件处理延迟、积压量
  - Stale index 导致的 false-positive 比例
  - Evict 事件丢失计数
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
