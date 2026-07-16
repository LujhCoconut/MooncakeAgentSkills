# MooncakeAgentSkills

> 🧠 从 Paper work 到快速落地工业级代码

[![Skill](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://claude.ai/code)
[![Target](https://img.shields.io/badge/Target-Mooncake-orange)](https://github.com/kvcache-ai/Mooncake)
[![License](https://img.shields.io/badge/License-LujhCoconut-green)](./LICENSE)

MooncakeAgentSkills 是一个 **Claude Code Skill**（安装为 `/mooncake-agent-skills`）。它与 [domain_knowledge_agent](https://github.com/LujhCoconut/domain_knowledge_agent) 联动，自动扫描 **[Mooncake](https://github.com/kvcache-ai/Mooncake)**（Moonshot AI 开源的 KVCache 中心化 LLM 推理服务平台，FAST 2025 Best Paper）的源代码，检索论文洞察，评估可应用性，生成结构化优化方案。

---

## 核心能力

| 能力 | 入口 | 说明 |
|------|------|------|
| **组件级优化分析** | `/mooncake-agent-skills "问题"` | 两级路由 → 源码分析 → 论文检索 → 方案生成 |
| **排队论建模** | `/mooncake-agent-skills "排队论分析"` | 输入硬件配置 → 14 个排队点自动估算延迟 → 瓶颈排名 |
| **快速问答** | `/mooncake-qa "问题"` | 概念 · 49 个配置参数 · 4 棵诊断树 · 11 个错误速查 |
| **三视角理论** | `mooncake/architecture.md` | 分布式存储 + 高性能通信 + LLM Serving 统一模型 |

---

## 安装

```bash
git clone git@github.com:LujhCoconut/MooncakeAgentSkills.git ~/.claude/skills/mooncake-agent-skills/

# 前置依赖
git clone git@github.com:LujhCoconut/domain_knowledge_agent.git ~/.claude/skills/domain-knowledge/
git clone https://github.com/kvcache-ai/Mooncake.git ~/src/Mooncake
```

> `MOONCAKE_REPO` 环境变量可自定义 Mooncake 源码路径，默认 `~/src/Mooncake`。

## 使用示例

### 组件级优化分析

```bash
# Transfer Engine
/mooncake-agent-skills "降低 RDMA 传输延迟"
/mooncake-agent-skills "TENT 多路径切片调度如何自适应链路质量"
/mooncake-agent-skills "GPU-NIC 拓扑发现是否可用于调度优化"

# Mooncake Store
/mooncake-agent-skills "改进 KV cache 驱逐策略，引入访问频率感知"
/mooncake-agent-skills "Master 元数据服务如何实现高可用"
/mooncake-agent-skills "G1→G2→G3 三级存储的迁移策略优化"

# Conductor · Integrations · Build · Operations
/mooncake-agent-skills "Conductor 前缀缓存索引结构优化"
/mooncake-agent-skills "pybind11 Python 绑定零拷贝路径验证"
/mooncake-agent-skills "端到端可观测性：metrics + tracing + logging"
```

### 排队论建模与延迟估算

提供硬件配置和工作负载，自动计算所有排队点延迟、定位瓶颈：

```bash
# 使用默认参数快速扫描
/mooncake-agent-skills "对 Mooncake 进行排队论分析"

# 带入具体硬件
/mooncake-agent-skills "排队论分析: ConnectX-7 400Gbps, 8×H100, Samsung PM9A3 NVMe×4,
                   QPS=100, avg_prompt=4000t, avg_decode=800t, model=Llama-70B"

# 特定场景
/mooncake-agent-skills "PD 分离场景下 Prefill→Decode KV cache 传输的排队延迟估算"
/mooncake-agent-skills "G3 NVMe SSD 层的队列深度和延迟瓶颈分析"
```

**排队论模块能力**：
- **硬件查表**：7 款 NIC（ConnectX-7→ConnectX-5→EFA→TCP）、5 款 NVMe SSD（P5800X Optane→PM9A3→通用）、4 款 GPU、4 个模型的 KV/token 大小
- **负载推导**：从 QPS/batch_size/hit_rate 自动算出 9 个到达率（λ_put, λ_get, λ_lookup...）
- **14 个排队点**：Q1-Q5 (Transfer Engine) + Q6-Q10 (Store) + Q11-Q13 (Conductor) + Q_G3 (NVMe I/O)
- **延迟瀑布图 + 瓶颈排名**：端到端路径上各排队点占比可视化和优化优先级

### 快速问答

概念解释、配置查询、故障排查：

```bash
/mooncake-qa "Mooncake 的 PD 分离是什么意思"
/mooncake-qa "怎么确认 GPUDirect RDMA 有没有生效"
/mooncake-qa "Master 挂了推理还能继续吗"
/mooncake-qa "RDMA 报 cannot create QP 错怎么办"
/mooncake-qa "Transfer Engine 的 MC_BATCH_SIZE 默认值是多少"
/mooncake-qa "G2 满了会怎样？驱逐检查间隔怎么调"
```

**QA 覆盖**：概念与架构 (5)、配置参数大全 (49 项)、环境安装 (4)、Transfer Engine/RDMA (3)、Store (5)、故障排查 (4 棵诊断树 + 11 个错误速查)、性能调优 (4)。

---

## Mooncake 组件覆盖

| 组件 | 目录 | 子组件数 | 覆盖要点 |
|------|------|:--------:|---------|
| **Transfer Engine / TENT** | `mooncake/transfer-engine/` | 4 | transport (RDMA/TCP/NVLink), tent (切片/选择/遥测), memory (注册/分配器/零拷贝), topology (GPU-NIC/NUMA/亲和性) |
| **Mooncake Store** | `mooncake/store/` | 4 | storage-backend (DRAM/SSD/三级存储), master (HA/租约/驱逐), client (读写/连接池/P2P), replication (副本/亲和性/拓扑放置) |
| **Conductor** | `mooncake/conductor/` | 1 | 哈希索引、前缀匹配、事件管线、多租户、Cache 命中率 |
| **Python Bindings** | `mooncake/integrations/` | 1 | pybind11 零拷贝、异步 store、内存分配器 |
| **Build & Deploy** | `mooncake/build-deploy/` | 1 | CMake 并行编译、CI/CD、Wheel 打包 |
| **Operations & SRE** | `mooncake/operations/` | 1 | 可观测性、故障恢复、基准测试 |
| **Queueing Theory** | `mooncake/queueing-theory/` | 1 | M/M/1 M/M/c M/G/1 模型、硬件查表、14 排队点延迟估算、瓶颈排名、4 场景分类器 |
| **Quick Q&A** | `qa/` | 1 | 49 配置参数、4 棵诊断树、11 错误速查 |

> 子组件各自维护 `SKILL.md`（优化维度 + 领域知识映射）和 `KNOWLEDGE.md`（累积优化目标）。每个 SKILL.md 末尾有维护规则：任何实质性更新需同步检查 `README.md`。

---

## 项目结构

```
MooncakeAgentSkills/
├── SKILL.md                     # /mooncake-agent-skills 入口 (路由 + pre/post git sync)
├── CLAUDE.md                    # 项目指令 (行为准则、扩展规范)
├── config.md                    # 仓库路径、远程地址、环境变量
├── LICENSE
├── README.md                    # 本文件
│
├── common/                      # 跨项目通用优化方法论
│   └── SKILL.md                 #   35+ 优化模式、6 步分析流程、6 维评估框架、反模式警示
│
├── mooncake/                    # Mooncake 专项
│   ├── architecture.md          #   **三视角统一模型** (分布式存储+通信+Serving，含 CAP/RDMA 模型/KV cache 语义)
│   ├── SKILL.md                 #   组件路由器 (两级路由 + Mooncake 源码同步)
│   ├── repo-map.md              #   源码树→子组件映射 (含 [tag] 标注)
│   ├── transfer-engine/         #   Transfer Engine / TENT
│   │   ├── SKILL.md             #     子路由器
│   │   ├── transport/           #     RDMA/TCP/NVLink/EFA/CXL 协议
│   │   ├── tent/                #     切片调度·动态选择·遥测·自愈
│   │   ├── memory/              #     内存注册·NVLink 分配器·零拷贝路径
│   │   └── topology/            #     GPU-NIC PCIe/NVLink/NUMA 拓扑
│   ├── store/                   #   Mooncake Store
│   │   ├── SKILL.md             #     子路由器
│   │   ├── storage-backend/     #     DRAM 分配·SSD 裸盘 (Direct I/O/SPDK)·G1/G2/G3 迁移
│   │   ├── master/              #     HA·Raft/ETCD·一致性·segment·lease·驱逐(价值感知)
│   │   ├── client/              #     读写路径·连接池·批量·P2P·RealClient/DummyClient
│   │   └── replication/         #     副本策略·拓扑感知·亲和/反亲和·动态副本数
│   ├── conductor/               #   KV Cache Indexer (前缀索引·命中率·事件管线·一致性)
│   ├── integrations/            #   Python 绑定 (pybind11 零拷贝·异步 store·框架连接器)
│   ├── build-deploy/            #   CMake 并行编译·CI/CD·Wheel 打包
│   ├── operations/              #   可观测性·故障恢复·Benchmark
│   └── queueing-theory/         #   **排队论建模** (M/M/1 M/M/c M/G/1·14 排队点·硬件查表·瓶颈排名)
│
├── qa/                          # Mooncake 快速问答
│   ├── SKILL.md                 #   /mooncake-qa 入口
│   └── KNOWLEDGE.md             #   49 配置参数 + 4 诊断树 + 11 错误速查
│
├── proposals/                   # 生成的优化方案
└── history/                     # 优化会话日志
```

---

## 与 domain_knowledge_agent 联动

```
domain_knowledge_agent                  MooncakeAgentSkills
┌────────────────────────┐             ┌──────────────────────────┐
│                        │             │                          │
│ 论文解析 · 洞察归档     │  论文洞察    │ 源码分析 · 方案生成       │
│                        │  ────────→  │                          │
│ SKILL.md (路由)         │             │ SKILL.md (路由+分析)      │
│ KNOWLEDGE.md (知识)     │  ←─ 新知 ─  │ KNOWLEDGE.md (优化目标)   │
│                        │   反馈       │                          │
│ 论文归档 + 阅读笔记      │             │ 优化提案 + Q&A + 排队论   │
└────────────────────────┘             └──────────────────────────┘
```

**三种交互**：直接文件读取 / Skill 工具调用 / 交叉引用。两个项目共享 SKILL.md / KNOWLEDGE.md 命名约定和 Git pre-op pull + post-op push 同步策略。

---

## 理论根基

### architecture.md — 三视角统一模型

在执行任何组件优化前，先理解 Mooncake 在三个视角中的位置：

| 视角 | 核心内容 |
|------|---------|
| **分布式存储系统** | CAP 分析、Immutable 对象简化一致性、控制/数据平面分离、G1/G2/G3 延迟模型、经典分布式问题映射 |
| **高性能通信系统** | RDMA 延迟模型、零拷贝前提、多路径 M×N 调度、协议选择的多目标优化、Memory pin 开销模型 |
| **LLM 推理 Serving** | KV cache 7 个独特语义、前缀缓存价值量化、PD 分离端到端延迟模型、优化杠杆优先级 |

### queueing-theory — 排队论建模引擎

输入硬件配置和工作负载 → 自动计算延迟瓶颈：

| 阶段 | 内容 |
|------|------|
| **场景识别** | 23 个参数提取 (硬件 12 + 负载 11)，4 种场景分类 (PD 分离/高 QPS/G3 频繁/Conductor 瓶颈) |
| **硬件查表** | NIC (7 款) → μ 值，NVMe (5 款) → IOPS/带宽/延迟，GPU (4 款) → 算力/HBM，模型 (4 个) → KV/token |
| **负载推导** | QPS × batch_size × hit_rate → λ_put, λ_get, λ_lookup, λ_renew, λ_evict, λ_conductor |
| **延迟估算** | 14 个排队点逐一 M/M/1 M/M/c M/G/1 计算，含具体数值示例 |
| **瓶颈排名** | 延迟瀑布图 + ρ 排序 + 优化建议 + 灵敏度分析 |

### common/SKILL.md — 通用优化方法论

35+ 优化模式 (数据通路/资源调度/存储缓存/并发并行/可靠性)，6 维评估，4 级可应用性评级。

---

## 维护规则

每个 `SKILL.md` 末尾均有：**任何实质性更新（新增优化维度、修改分析流程、更新路由规则等），必须同步更新本 README.md** — 确保文档始终反映最新能力边界。

## License

Copyright (c) 2026 LujhCoconut
