# Mooncake Conductor (KV Cache Indexer) Optimization

Conductor 是 Mooncake 的 KV Cache 前缀感知路由服务。它订阅 Store 中的 KV cache 事件，基于 XXH3-64 滚动哈希建立前缀缓存索引，并通过 HTTP API 提供缓存命中查询。

## 源代码地图

参见 `mooncake/repo-map.md` § Conductor。

> **注意**: Conductor 的核心代码可能位于独立仓库或尚未在 Mooncake 主仓库完全开源。分析时以 `docs/design/conductor/` 中的设计文档为主要参考。如有本地 Conductor 仓库，请在 config.md 中添加路径。

### 设计文档
- `docs/design/conductor/conductor-architecture-design.md` — 架构总览
- `docs/design/conductor/indexer-api-design.md` — Indexer API 设计

### 关键组件（基于设计文档）
- `EventManager` — 订阅和分发 KV cache 事件
- `KVEventHandler` — 事件处理，更新前缀缓存表
- `PrefixCacheTable` — XXH3-64 滚动哈希索引
- `ModelContext` — 多模型、多租户上下文隔离

## 优化维度

### 1. 哈希函数与索引结构
- **设计参考**: `indexer-api-design.md`
- **当前状态**: XXH3-64 滚动哈希，`/query` (token IDs) 和 `/query_by_hash` (precomputed hash) 两种查询
- **关键问题**:
  - XXH3-64 的选择依据？64-bit 是否有碰撞风险？
  - 滚动哈希的开销？（每个新 token 滚动一次）
  - 索引结构是否支持范围查询？（前缀长度 ≥ N）
- **Domain Knowledge 映射**:
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 哈希算法选择
  - `performance/gpu-ai-performance/KNOWLEDGE.md` — KV cache 优化

### 2. 前缀匹配算法
- **设计参考**: `PrefixCacheTable` 的设计
- **当前状态**: 基于滚动哈希的前缀匹配
- **关键问题**:
  - 最长前缀匹配 (Longest Prefix Match) 的查找复杂度？
  - 是否支持模糊匹配？（允许部分 token 不匹配）
  - 缓存表的空间占用和压缩？
- **Domain Knowledge 映射**:
  - `algorithms/graph-processing/KNOWLEDGE.md` — 前缀树/Radix Tree
  - `performance/optimization-paradigms/KNOWLEDGE.md` — 空间优化

### 3. 事件处理管线
- **设计参考**: `EventManager`, `KVEventHandler`
- **当前状态**: 订阅 Store 事件 (Put/Evict/Update)，ZMQ 通信
- **关键问题**:
  - 事件处理的吞吐和延迟？（高写入频率下是否 lag？）
  - 事件处理的顺序保证？（FIFO? 乱序容忍？）
  - 事件丢失后的恢复机制？
- **Domain Knowledge 映射**:
  - `performance/optimization-paradigms/KNOWLEDGE.md` — 事件驱动架构
  - `operations/monitoring-observability/KNOWLEDGE.md` — 流水线监控

### 4. 缓存失效策略
- **设计参考**: Conductor 与 Store 的交互
- **当前状态**: 被动接收 Evict 事件更新索引
- **关键问题**:
  - Evict 事件到索引更新的延迟？
  - 是否有主动失效机制？（TTL-based？）
  - 不一致窗口内的错误路由影响？
- **Domain Knowledge 映射**:
  - `architecture/cloud-native/KNOWLEDGE.md` — 缓存一致性
  - `algorithms/distributed-consensus/KNOWLEDGE.md` — 状态同步

### 5. 多租户路由
- **设计参考**: `ModelContext` = (tenant_id, model_name, lora_name, block_size, salt, instance_id)
- **当前状态**: 按 ModelContext 隔离
- **关键问题**:
  - 按租户隔离的索引是否导致内存膨胀？
  - 是否支持跨租户的 cache sharing？（相同 base model）
  - salt 的设计用途？（安全？隔离？）
- **Domain Knowledge 映射**:
  - `algorithms/distributed-consensus/KNOWLEDGE.md` — 多租户架构
  - `security/os-security/KNOWLEDGE.md` — 租户隔离安全

## 分析工作流

1. 阅读设计文档理解架构意图
2. 如有代码仓库，搜索关键类实现
3. 检索相关领域知识
4. 生成优化方案

## 已知优化目标

参见 `KNOWLEDGE.md`。
