# Mooncake Transfer Engine / TENT — 子组件路由

Transfer Engine 是 Mooncake 的数据平面，负责多协议、多 GPU 厂商的点对点数据传输。TENT (Transfer Engine NEXT) 是下一代运行时。本文件根据优化问题关键词分发到 4 个子组件。

## Transfer Engine 架构速览

```
┌───────────────────────────────────────────────────────────┐
│                   Transfer Engine                          │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ transport │  │   tent   │  │  memory  │  │ topology  │ │
│  │ 传输协议  │  │ TENT调度 │  │ 内存管理  │  │ 拓扑发现  │ │
│  │ RDMA·TCP │  │切片·选择 │  │注册·分配 │  │GPU-NIC·NUMA│ │
│  │ NVLink·  │  │遥测·配置 │  │零拷贝·   │  │           │ │
│  │ EFA·CXL  │  │          │  │NVLink    │  │           │ │
│  └──────────┘  └──────────┘  └──────────┘  └───────────┘ │
└───────────────────────────────────────────────────────────┘
```

## 子组件路由表

| 子组件 | 目录 | 职责 | 匹配关键词 |
|--------|------|------|-----------|
| **transport** | `transport/` | 各传输协议实现 (RDMA/TCP/NVLink/EFA/CXL/NVMe-oF)、协议选择、多协议混合、拥塞控制 | rdma, tcp, nvlink, efa, cxl, nvmeof, 协议, protocol, transport, 传输, congestion, 拥塞, DCQCN, multi-rail, 多轨, GPUDirect, GDR, inline, message size, QP, queue pair, roce, infiniband, 带宽, bandwidth, latency, 延迟 |
| **tent** | `tent/` | TENT 运行时：切片调度器、动态传输选择、遥测收集、配置管理、基准测试 | tent, 切片, slice, scheduler, 调度器, transport selector, 传输选择, telemetry, 遥测, 自适应, adaptive, declarative, 声明式, 多路径, multi-path, 自愈, self-healing, qos |
| **memory** | `memory/` | 内存注册 (RDMA pin)、NVLink 分配器、零拷贝路径、GPU 内存管理、跨 NUMA 访问 | memory, 内存, register, 注册, pin, allocator, 分配器, nvlink-allocator, zero-copy, 零拷贝, gpu memory, 显存, vram, hbm, cuda, managed buffer, dma, numa, 跨节点 |
| **topology** | `topology/` | GPU-NIC 拓扑发现、NUMA 感知、PCIe 拓扑、NVLink 拓扑、亲和性绑定 | topology, 拓扑, gpu-nic, pcie, numa, affinity, 亲和性, 绑定, bind, discovery, 发现, bus, interconnect, 互连 |

## 路由逻辑

1. **单关键词匹配**：在用户问题中搜索上表关键词，匹配最多的子组件
2. **传输协议问题**：大概率路由到 `transport` + `tent`（TENT 负责协议选择）
3. **端到端延迟/吞吐优化**：可能涉及全部 4 个子组件
4. **跨组件**：如果问题涉及 Transfer Engine 与 Store 的交互，同时路由到 `store/client/`

## 各子组件入口

| 子组件 | SKILL.md | 主要源码 (参见 repo-map.md) |
|--------|----------|--------------------------|
| transport | `transport/SKILL.md` | `src/transport/` (8 个协议实现) |
| tent | `tent/SKILL.md` | `tent/runtime/`, `tent/config/`, `tent/benchmark/` |
| memory | `memory/SKILL.md` | `memory_location.cpp`, `nvlink-allocator/` |
| topology | `topology/SKILL.md` | `topology.cpp` |

## 分析流程

同 Store 路由的流程。详见 `../../common/SKILL.md`。


## ⚠️ 维护规则

**当本 SKILL.md 有实质性更新（新增优化维度、修改分析流程、更新路由规则、新增领域知识映射等），必须同步检查并更新 `README.md` 中对应的项目结构、组件覆盖表、使用示例或模块说明。** 保持 README 始终反映最新的 skill 能力边界。
