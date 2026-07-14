# Mooncake Operations — 已知优化目标

本文档记录运维与 SRE 相关的优化机会（跨组件）。

> **最后更新**: 2026-07-14
> **已分析次数**: 0 (初始种子文件)

---

## 可观测性增强（初始分析）

### 当前状态
- `transfer_engine_topology_dump` CLI 工具
- TENT telemetry 收集
- 缺乏统一的 metrics 导出和告警框架

### 潜在优化方向
- 集成 OpenTelemetry / Prometheus metrics 导出
- 关键 RED metrics（Rate, Errors, Duration）仪表盘
  - Transfer Engine: submit 速率、传输延迟分布、错误率
  - Store: put/get 速率、延迟、命中率、驱逐率
  - Master: segment 利用率、租约过期率
- 分布式 tracing（端到端请求的 trace 传播）
- 结构化日志（JSON format，统一字段）

### 相关领域知识
- `operations/monitoring-observability/KNOWLEDGE.md`
- `operations/cloud-infrastructure/KNOWLEDGE.md`

---

## 故障恢复流程优化（初始分析）

### 当前状态
- 各组件独立的错误处理
- RDMA 链路断开需要手动或超时恢复
- Master 依赖外部 metadata store 的 HA

### 潜在优化方向
- RDMA 链路的主动健康检查（keepalive / heartbeat）
- 连接断开到客户端感知的延迟优化（快速失败 vs. 等待超时）
- Master failover 的客户端透明重连
- Graceful degradation: 当 RDMA 不可用时自动降级到 TCP
- 故障注入测试（chaos engineering）验证恢复流程

### 相关领域知识
- `operations/cloud-infrastructure/KNOWLEDGE.md`
- `common/diagnosis-playbooks/`

---

## 基准测试体系完善（初始分析）

### 当前状态
- `transfer_engine_bench`, `tent_bench`, `store_bench`
- 需进一步了解覆盖范围和可重复性

### 潜在优化方向
- 标准化 benchmark 配置集（small/medium/large payload, 单节点/多节点）
- 建立 performance baseline 数据库（per commit 的持续性能追踪）
- 添加 micro-benchmark 覆盖关键路径（内存注册、submit、poll）
- 端到端 benchmark（模拟真实推理场景的 KV cache 读写模式）

### 相关领域知识
- `performance/profiling-methodology/KNOWLEDGE.md`
- `operations/os-performance-tuning/KNOWLEDGE.md`
