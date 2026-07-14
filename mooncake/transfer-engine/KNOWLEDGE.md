# Mooncake Transfer Engine — 已知优化目标

本文档记录 Transfer Engine 组件中已识别的优化机会。每次优化分析后如有新发现，在此追加。

> **最后更新**: 2026-07-14
> **已分析次数**: 0 (初始种子文件)

---

<!-- 以下为种子优化目标，基于公开文档和论文知识的初步分析 -->

## RDMA 传输延迟优化（初始分析）

### 当前状态
- RDMA 传输实现在 `src/transport/rdma_transport.cpp`
- 使用单 QP (Queue Pair) 或静态多 QP 配置
- 需进一步分析：是否使用 GPUDirect RDMA、是否多 rail

### 潜在优化方向
- 多 rail RDMA 的负载均衡
- 拥塞控制（DCQCN 调优）
- RDMA 内存 pin 的缓存和复用策略
- 自适应 message size 切换（small message → inline, large → RDMA write）

### 相关领域知识
- `network/os-networking/KNOWLEDGE.md` — RDMA 与网络协议栈优化
- `performance/network-performance/KNOWLEDGE.md` — 网络性能调优

---

## 批量传输调度优化（初始分析）

### 当前状态
- `submitTransfer()` 支持批量提交
- 需进一步分析：内部分批/合并策略、调度优先级

### 潜在优化方向
- 自适应批量大小（基于传输大小和链路状态）
- 优先级调度（区分延迟敏感和吞吐优先传输）
- 批量内乱序执行的完成通知优化

### 相关领域知识
- `algorithms/resource-scheduling/KNOWLEDGE.md` — 调度算法
- `performance/optimization-paradigms/KNOWLEDGE.md` — 批处理优化

---

## TENT 切片调度优化（初始分析）

### 当前状态
- TENT 处于 Phase 1（2026 Q1），切片调度器基础实现
- Phase 2 规划中：声明式切片调度、链路自愈、拓扑感知图

### 潜在优化方向
- 切片大小自适应（基于 MTU、RTT、拥塞窗口）
- 多路径拥塞感知的权重分配
- 片间依赖感知的调度（避免 head-of-line blocking）
- 传输失败时的局部重传（vs. 全量重传）

### 相关领域知识
- `algorithms/resource-scheduling/KNOWLEDGE.md`
- `network/os-networking/KNOWLEDGE.md`

---

<!-- 后续优化分析发现的目标在此追加 -->
<!-- 格式参考 domain_knowledge_agent 的 KNOWLEDGE.md 三段式结构 -->
