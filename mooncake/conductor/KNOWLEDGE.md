# Mooncake Conductor — 已知优化目标

本文档记录 Conductor (KV Cache Indexer) 组件中已识别的优化机会。

> **最后更新**: 2026-07-14
> **已分析次数**: 0 (初始种子文件)

---

## 前缀缓存索引结构优化（初始分析）

### 当前状态
- XXH3-64 滚动哈希，支持 token IDs 和预计算哈希两种查询
- 按 ModelContext 隔离的多租户索引

### 潜在优化方向
- 评估碰撞概率：64-bit 在亿级 key 下的碰撞率，是否需要 128-bit 或双哈希
- 引入 Radix Tree / Adaptive Radix Tree (ART) 用于最长前缀匹配
- 压缩前缀表（如使用 minimal perfect hashing）
- 分层索引：热门前缀 → L1 cache，全量 → L2

### 相关领域知识
- `algorithms/resource-scheduling/KNOWLEDGE.md`
- `algorithms/graph-processing/KNOWLEDGE.md`

---

## 事件处理吞吐优化（初始分析）

### 当前状态
- ZMQ 接收 Store 事件，单线程或有限并发的 event handler
- 事件类型：Put / Evict / Update

### 潜在优化方向
- 事件批处理（batch 窗口内的多个事件合并处理）
- 无锁事件队列（SPSC / MPMC ring buffer）
- 根据事件类型做优先级处理（Put vs. Evict）
- 异步索引更新（允许短暂不一致换取吞吐）

### 相关领域知识
- `performance/optimization-paradigms/KNOWLEDGE.md`
- `algorithms/concurrent-data-structures/KNOWLEDGE.md`

---

## 缓存命中率提升（初始分析）

### 当前状态
- 前缀感知路由，基于精确 token 序列匹配

### 潜在优化方向
- 支持编辑距离 ≤ k 的近似匹配
- 语义级前缀匹配（system prompt 的语义 hash）
- 多轮对话的跨轮次缓存复用
- 预热策略：高频请求的主动索引预加载

### 相关领域知识
- `performance/gpu-ai-performance/KNOWLEDGE.md` — LLM 推理 KV cache
- `algorithms/resource-scheduling/KNOWLEDGE.md`
