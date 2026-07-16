# Mooncake Transfer Engine — TENT 已知优化目标

> **最后更新**: 2026-07-16
> **已分析次数**: 0

---

## 切片大小自适应（初始分析）

### 当前状态
- Phase 1 切片调度，Phase 2 声明式调度
- 需验证切片大小的决策逻辑

### 潜在优化方向
- 基于 RTT 和带宽的切片大小动态调整（bandwidth-delay product 计算）
- 不同路径用不同切片大小（高速路径大切片，慢速路径小切片）
- 切片大小与 MTU 对齐（避免 IP fragmentation）
- 历史传输数据学习最优切片大小

### 相关领域知识
- `algorithms/resource-scheduling/KNOWLEDGE.md`
- `network/os-networking/KNOWLEDGE.md`

---

## 拥塞感知的多路径权重（初始分析）

### 当前状态
- 多路径切片调度基础实现
- 需验证是否有拥塞信号作为输入

### 潜在优化方向
- 实时拥塞信号收集（ECN、RTT 变化、completion queue depth）
- 拥塞感知的权重调整算法（类似 MPTCP 的耦合拥塞控制）
- 路径故障时的快速权重清零和恢复时的渐进提升

### 相关领域知识
- `network/os-networking/KNOWLEDGE.md`
- `performance/optimization-paradigms/KNOWLEDGE.md`

---

## 链路自愈与 QoS（Phase 2 前瞻）

### 当前状态
- Phase 2 规划中

### 潜在优化方向
- 链路故障的毫秒级检测（heartbeat + 硬件中断）
- 局部重传（只重传失败的切片，不重传整个传输）
- 声明式 QoS：用户声明 "p99 < 1ms"，调度器自动选择协议和切片策略
- 拓扑感知的统一图模型（NUMA/PCIe/RDMA/NVLink 统一建模为加权图）

### 相关领域知识
- `operations/cloud-infrastructure/KNOWLEDGE.md`
- `algorithms/resource-scheduling/KNOWLEDGE.md`
