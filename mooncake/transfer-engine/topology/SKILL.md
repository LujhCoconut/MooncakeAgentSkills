# Mooncake Transfer Engine — Topology 拓扑发现优化

拓扑发现负责检测 GPU-NIC 的物理连接关系——PCIe 层级、NVLink 拓扑、NUMA topology。从通信系统视角看，拓扑信息是上层所有调度决策的基础——没有准确的拓扑，协议选择、切片分配、内存放置都是盲目的。

> **理论根基**: 参见 `../architecture.md` §3.5 (通信系统优化维度)、§3.1 (协议选择需要拓扑信息)、附录 (RDMA 零拷贝的拓扑前提)

## 通信系统视角

### 拓扑在通信系统中的角色

```
┌─────────────────────────────────────────┐
│             应用层 (调度/路由)            │ ← 使用拓扑信息的消费者
├─────────────────────────────────────────┤
│           Topology Service              │ ← 拓扑发现与查询
├─────────────────────────────────────────┤
│  PCIe  │  NUMA  │ NVLink │ InfiniBand  │ ← 各层拓扑
├─────────────────────────────────────────┤
│            硬件层                        │
└─────────────────────────────────────────┘

拓扑服务的 API:
  get_distance(src, dst) → 加权距离 (综合延迟、带宽、跳数)
  get_shortest_path(src, dst) → 最优路径
  get_affinity_group(entities) → 亲和性分组
  get_cross_numa_penalty(numa_a, numa_b) → 延迟/带宽惩罚
  is_gdr_capable(gpu, nic) → GDR 是否可用
```

### 拓扑信息质量 = 调度决策质量

错误的拓扑信息导致次优决策：

| 如果拓扑认为... | 实际... | 后果 |
|---------------|---------|------|
| GPU 0 ↔ NIC 0 是 local | 它们在不同的 PCIe root complex | RDMA 延迟翻倍 |
| NVLink 链路是 x16 | 实际降级到 x8 | 带宽估算偏高 50% |
| NUMA node 0 和 1 延迟 ~100ns | 实际 ~300ns | 内存放置错误 |
| 所有节点同构 | GPU 数量/NIC 数不同 | 负载均衡失效 |

## 源代码地图

| 文件 | 说明 | 子组件 |
|------|------|--------|
| `mooncake-transfer-engine/src/topology.cpp` | GPU-NIC 拓扑发现 | topology |
| `mooncake-transfer-engine/include/transfer_engine.h` | getTopology() 等接口 | topology |
| CLI: `transfer_engine_topology_dump` | 拓扑 dump 工具 | topology |

## 优化维度

### 1. GPU-NIC 拓扑发现的精度与粒度

- **代码入口**: `topology.cpp`

**PCIe topology 示例**：

```
Socket 0 (NUMA node 0):
├── PCIe Root Complex 0
│   ├── PCIe Switch 0
│   │   ├── GPU 0 (0000:17:00.0)    ← BDF (Bus:Device.Function)
│   │   └── GPU 1 (0000:1a:00.0)
│   └── PCIe Switch 1
│       ├── NIC 0 (mlx5_0, 0000:51:00.0)
│       └── NVMe 0 (0000:54:00.0)
│
Socket 1 (NUMA node 1):
├── PCIe Root Complex 1
    ├── GPU 2 (0000:b1:00.0)
    ├── GPU 3 (0000:b4:00.0)
    └── NIC 1 (mlx5_1, 0000:d1:00.0)
```

**拓扑发现需要回答的问题**：
- GPU X 和 NIC Y 是否在同一 PCIe root complex? → GDR 可行性
- GPU X 和 GPU Y 是否通过 NVSwitch 连接? → P2P 带宽
- NIC Y 在哪个 NUMA node? → 中断亲和性
- NVMe 0 在哪个 PCIe switch? → 存储带宽不与其他设备竞争

**关键问题**:
- 拓扑发现的粒度和准确性？（是否深入到 PCIe switch 层级？）
- 是否支持 NVLink/NVSwitch topology？
- 节点异构性如何表示？（同一集群中不同配置的节点）
- 拓扑发现是启动时一次性还是支持动态刷新？

**Domain Knowledge 映射**:
- `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU 通信拓扑
- `architecture/accelerators/KNOWLEDGE.md` — 硬件拓扑、PCIe

### 2. NUMA 感知调度 — 从拓扑到决策

- **代码入口**: 调度器如何使用拓扑信息

**NUMA 感知的调度决策**：

| 决策 | 使用什么拓扑信息 | 错误的代价 |
|------|----------------|-----------|
| **CPU core 绑定** | NUMA node of NIC | 中断跨 NUMA 转发 (2×延迟) |
| **内存分配** | NUMA node of 请求方 | 跨 NUMA 内存访问 (1.5-2×延迟) |
| **GPU 选择** | NUMA node of 调度 CPU | GPU kernel launch 跨 NUMA (可忽略, 但数据传输慢) |
| **NIC 选择** | NUMA node + GDR capability | 非 GDR 路径走 bounce buffer |

**理想的调度器**：

```
schedule(task, topology):
  # 找出所有满足硬件需求的 (CPU, GPU, NIC) 组合
  candidates = topology.find_candidates(task.constraints)

  # 按 NUMA 亲和性打分排序
  ranked = sort(candidates, key=lambda c:
      w1 * topology.numa_distance(task.requester, c.cpu) +
      w2 * topology.numa_distance(c.gpu, c.nic) +
      w3 * topology.pcie_distance(c.gpu, c.nic))

  return ranked[0]
```

**关键问题**:
- 当前调度器是否使用了完整的 NUMA 拓扑信息？
- CPU core 是否绑定到 NIC 所在的 NUMA node？
- 内存分配是否优先在 local NUMA node？
- 是否有跨 NUMA 操作的统计/告警？

**Domain Knowledge 映射**:
- `performance/system-tuning/KNOWLEDGE.md` — NUMA tuning, IRQ affinity
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — NUMA architecture

### 3. 拓扑变更检测 — 动态环境

- **关键问题**: 生产环境中拓扑不是静态的

**拓扑变更场景**：

| 场景 | 影响 | 检测方式 |
|------|------|---------|
| **GPU 故障** | 可用的 GPU memory 减少 | Driver event / nvidia-smi |
| **NIC 降级** | 链路速度从 200G → 100G | ethtool / driver event |
| **NVLink 降级** | 带宽从 x16 → x8 | nvidia-smi nvlink |
| **PCIe 热插拔** | 新设备加入/移除 | ACPI event |
| **节点故障** | 整个节点不可用 | Heartbeat / health check |

**变更后的恢复策略**：
- 拓扑信息的及时刷新 (不依赖重启)
- 已排队的传输任务如何处理？(快速失败 vs. 等待恢复)
- 调度器重新计算最优路径

**关键问题**:
- 拓扑发现是否支持动态刷新？
- 拓扑变更的检测延迟？
- 变更事件是否推送到调度器还是轮询？
- 部分降级 (链路降级但未完全断开) 的检测？

**Domain Knowledge 映射**:
- `operations/cloud-infrastructure/KNOWLEDGE.md` — 拓扑变更、故障检测
- `operations/monitoring-observability/KNOWLEDGE.md` — 硬件监控

### 4. 拓扑信息在 TENT 中的集成

- **TENT Phase 2 规划**: 拓扑感知的统一图模型

**统一拓扑图为加权有向图**：

```
Nodes: GPU, NIC, CPU socket, NVMe, NVSwitch
Edges: 物理连接 (PCIe, NVLink, QPI/UPI, ...)

边权重:
  weight(edge) = f(bandwidth, latency, is_cross_numa)

最短路径 (GPU_A → NIC_B):
  = Dijkstra(topology_graph, GPU_A, NIC_B)
  = GPU_A → [PCIe edge] → NIC_B  (如果同 PCIe root complex)
  = GPU_A → [QPI edge] → [PCIe edge] → NIC_B  (跨 NUMA, 权重更高)
```

**LLM 推理场景下的拓扑优化**：
- Prefill node 和 Decode node 之间的路径权重直接影响 PD 分离的延迟
- 如果 Prefill 和 Decode 在同一 NUMA domain 内，使用 SHM 可能比 RDMA 更快
- NVLink 直连的 GPU pair 应该优先用于高频的 KV cache 传输

**关键问题**:
- TENT 的拓扑图模型是如何表示的？（邻接矩阵？adjacency list？）
- 是否包含所有硬件层级的拓扑信息？
- 拓扑查询的延迟？（是否能满足实时调度的需求？）

**Domain Knowledge 映射**:
- `algorithms/graph-processing/KNOWLEDGE.md` — 图算法、最短路径
- `architecture/accelerators/KNOWLEDGE.md` — interconnect topology

### 5. Topology 可观测性

- **关键指标**:
  - 拓扑图的结构化 dump (JSON/Graphviz 格式)
  - 跨 NUMA 传输的比例、延迟惩罚
  - GPU-NIC affinity 的匹配率（local vs. remote）
  - 拓扑变更事件（时间、类型、影响）
  - NVLink/PCIe 链路速度和错误率
  - GDR-capable 的 (GPU, NIC) pair 数量
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
