# Mooncake Transfer Engine — Topology 已知优化目标

> **最后更新**: 2026-07-16
> **已分析次数**: 0

---

## 拓扑感知的 GPU-NIC 绑定（初始分析）

### 当前状态
- `topology.cpp` 发现 GPU-NIC 拓扑
- 需验证拓扑信息是否在调度决策中使用

### 潜在优化方向
- GPU 与 local NIC 的自动绑定（避免跨 NUMA node 的 PCIe 通信）
- NVLink 拓扑下的 GPU pair 优选（邻近 GPU 直连，减少 NVSwitch 跳数）
- 拓扑信息在 TENT transport selector 中的集成
- 异构拓扑的归一化表示（不同节点 GPU/NIC 配置不同时统一建模）

### 相关领域知识
- `performance/gpu-ai-performance/KNOWLEDGE.md`
- `architecture/accelerators/KNOWLEDGE.md`

---

## NUMA 感知的调度决策（初始分析）

### 当前状态
- 需验证调度器是否使用 NUMA 拓扑信息

### 潜在优化方向
- CPU core 与 NIC 的 NUMA 绑定（中断亲和性）
- NUMA 距离矩阵在放置决策中的使用
- 内存分配优先在访问方所在的 NUMA node
- 量化并暴露 NUMA 延迟惩罚（API 层面让上层应用感知）

### 相关领域知识
- `performance/system-tuning/KNOWLEDGE.md`
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md`

---

## 拓扑动态变更处理（初始分析）

### 当前状态
- 拓扑在启动时发现，需验证是否支持动态刷新

### 潜在优化方向
- 拓扑变更事件（GPU/NIC 热插拔、链路降级）的实时检测
- 变更后的调度策略平滑切换
- NVLink 链路降级（x16→x8→x4）的检测和告警
- 拓扑版本管理和变更日志

### 相关领域知识
- `operations/cloud-infrastructure/KNOWLEDGE.md`
- `operations/monitoring-observability/KNOWLEDGE.md`
