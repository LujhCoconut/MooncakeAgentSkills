# Mooncake Transfer Engine — Topology 拓扑发现优化

拓扑发现负责检测 GPU-NIC 的物理连接关系（PCIe 拓扑、NVLink 拓扑、NUMA 拓扑），为上层调度提供硬件感知能力。

## 源代码地图

| 文件 | 说明 |
|------|------|
| `mooncake-transfer-engine/src/topology.cpp` | GPU-NIC 拓扑发现 |
| `mooncake-transfer-engine/include/transfer_engine.h` | getTopology() 等接口 |
| `transfer_engine_topology_dump` CLI | 拓扑信息 dump 工具 |
| `mooncake-transfer-engine/src/transfer_engine.cpp` | transfer 调度中使用拓扑信息 |

## 优化维度

### 1. GPU-NIC 拓扑发现
- **代码入口**: `topology.cpp`
- **当前状态**: 支持拓扑发现和 dump
- **关键问题**:
  - 拓扑发现的粒度和准确性？（PCIe switch 层级？NVSwitch 拓扑？）
  - 拓扑信息是否在调度决策中使用？
  - 拓扑发现的性能开销？（启动时一次性 vs. 定期刷新？）
  - 是否支持异构拓扑？（不同节点 GPU/NIC 配置不同）
- **Domain Knowledge 映射**:
  - `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU 通信拓扑
  - `architecture/accelerators/KNOWLEDGE.md` — 硬件拓扑

### 2. NUMA 感知调度
- **代码入口**: 调度器如何使用拓扑信息
- **关键问题**:
  - 传输任务是否绑定到 local NUMA node 的 CPU core？
  - GPU-NIC 的 NUMA 亲和性是否影响 RDMA 延迟？
  - 内存分配是否优先在 local NUMA node？
  - 跨 NUMA 的传输延迟惩罚有多大？如何量化？
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — NUMA 优化
  - `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 内存层次

### 3. 拓扑变更检测
- **关键问题**:
  - GPU/NIC 热插拔的支持？
  - 节点故障时的拓扑更新？
  - NVLink 链路降级的检测？（x16→x8）
  - 拓扑变更后的调度策略调整？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md`
  - `architecture/cloud-native/KNOWLEDGE.md`

### 4. 多 GPU 拓扑优化
- **关键问题**:
  - NVSwitch 拓扑下的最优 GPU pair 选择？
  - 跨 NUMA GPU 的 P2P 传输路径？
  - GPU-NIC bonding（多 NIC 绑定到同一 GPU）的拓扑约束？
- **Domain Knowledge 映射**:
  - `performance/gpu-ai-performance/KNOWLEDGE.md`
  - `architecture/accelerators/KNOWLEDGE.md`

### 5. Topology 可观测性
- **关键指标**:
  - 拓扑图的结构化 dump（JSON/Graphviz）
  - 跨 NUMA 传输的比例和延迟
  - 拓扑变更事件日志
  - GPU-NIC 链路速度的监控
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
