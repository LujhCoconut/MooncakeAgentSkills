# MooncakeAgentSkills

> 🧠 从 Paper work 到快速落地工业级代码

[![Skill](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://claude.ai/code)
[![Target](https://img.shields.io/badge/Target-Mooncake-orange)](https://github.com/kvcache-ai/Mooncake)
[![License](https://img.shields.io/badge/License-LujhCoconut-green)](./LICENSE)

MooncakeAgentSkills 是一个 **Claude Code Skill**（安装为 `/mooncake-agent-skills`）。它与 [domain_knowledge_agent](https://github.com/LujhCoconut/domain_knowledge_agent) 联动，自动扫描 **[Mooncake](https://github.com/kvcache-ai/Mooncake)**（Moonshot AI 开源的 KVCache 中心化 LLM 推理服务平台，FAST 2025 Best Paper）的源代码，检索论文洞察，评估可应用性，生成结构化优化方案。同时内置 GitHub PR 审查和本地代码审查能力。

---

## 核心能力

| 能力 | 入口 | 说明 |
|------|------|------|
| **组件级优化分析** | `/mooncake-agent-skills optimize "问题"` | 两级路由 → 源码分析 → 论文检索 → 方案生成 |
| **排队论建模** | `/mooncake-agent-skills optimize "排队论分析"` | 输入硬件配置 → 14 个排队点自动估算延迟 → 瓶颈排名 |
| **GitHub PR 审查** | `/mooncake-agent-skills code-review [PR]` | 5 并行 agent + 置信度打分 + `gh` 回帖 |
| **本地代码审查** | `/mooncake-agent-skills review [aspects]` | git diff 多维度分析（含 perf-claims 收益真实性）→ 结构化报告 |
| **快速问答** | `/mooncake-agent-skills qa "问题"` | 概念 · 49 个配置参数 · 4 棵诊断树 · 11 个错误速查（回答首行标注「由AI总结，仅供参考！」） |
| **Q&A 入库** | `/mooncake-agent-skills update-qa "<文本>"` | 长文本 → 提取 Q&A → 源码验证（调用链 ±1 层 + 对抗反驳）→ 确认后入库；`audit` 模式重验已有条目防腐 |

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

> 输入 `/mooncake-agent-skills` 时，终端补全菜单显示参数提示 `[optimize|plan-feature|code-review|review|qa|update-qa|clear-proposals] <参数>`（来自根 SKILL.md frontmatter 的 `argument-hint`）。子命令级提示（如 `clear-proposals [today|month|year]`、`review [all|comments|tests|errors|types|code|simplify]`）在提交后通过交互式问答（AskUserQuestion）递归呈现，格式与各子目录 SKILL.md 的 `argument-hint` 保持一致。

### 组件级优化分析

```bash
# Transfer Engine
/mooncake-agent-skills optimize "降低 RDMA 传输延迟"
/mooncake-agent-skills optimize "TENT 多路径切片调度如何自适应链路质量"
/mooncake-agent-skills optimize "GPU-NIC 拓扑发现是否可用于调度优化"

# Mooncake Store
/mooncake-agent-skills optimize "改进 KV cache 驱逐策略，引入访问频率感知"
/mooncake-agent-skills optimize "Master 元数据服务如何实现高可用"
/mooncake-agent-skills optimize "G1→G2→G3 三级存储的迁移策略优化"

# Conductor · Integrations · Build · Operations
/mooncake-agent-skills optimize "Conductor 前缀缓存索引结构优化"
/mooncake-agent-skills optimize "pybind11 Python 绑定零拷贝路径验证"
/mooncake-agent-skills optimize "端到端可观测性：metrics + tracing + logging"
```

### 排队论建模与延迟估算

提供硬件配置和工作负载，自动计算所有排队点延迟、定位瓶颈：

```bash
# 使用默认参数快速扫描
/mooncake-agent-skills optimize "对 Mooncake 进行排队论分析"

# 带入具体硬件
/mooncake-agent-skills optimize "排队论分析: ConnectX-7 400Gbps, 8×H100, Samsung PM9A3 NVMe×4,
                   QPS=100, avg_prompt=4000t, avg_decode=800t, model=Llama-70B"

# 特定场景
/mooncake-agent-skills optimize "PD 分离场景下 Prefill→Decode KV cache 传输的排队延迟估算"
/mooncake-agent-skills optimize "G3 NVMe SSD 层的队列深度和延迟瓶颈分析"
```

**排队论模块能力**：
- **硬件查表**：7 款 NIC（ConnectX-7→ConnectX-5→EFA→TCP）、5 款 NVMe SSD（P5800X Optane→PM9A3→通用）、4 款 GPU、4 个模型的 KV/token 大小
- **负载推导**：从 QPS/batch_size/hit_rate 自动算出 9 个到达率（λ_put, λ_get, λ_lookup...）
- **14 个排队点**：Q1-Q5 (Transfer Engine) + Q6-Q10 (Store) + Q11-Q13 (Conductor) + Q_G3 (NVMe I/O)
- **延迟瀑布图 + 瓶颈排名**：端到端路径上各排队点占比可视化和优化优先级

### 快速问答

概念解释、配置查询、故障排查：

```bash
/mooncake-agent-skills qa "Mooncake 的 PD 分离是什么意思"
/mooncake-agent-skills qa "怎么确认 GPUDirect RDMA 有没有生效"
/mooncake-agent-skills qa "Master 挂了推理还能继续吗"
/mooncake-agent-skills qa "RDMA 报 cannot create QP 错怎么办"
/mooncake-agent-skills qa "Transfer Engine 的 MC_BATCH_SIZE 默认值是多少"
/mooncake-agent-skills qa "G2 满了会怎样？驱逐检查间隔怎么调"
```

**QA 覆盖**：概念与架构 (5)、配置参数大全 (49 项)、环境安装 (4)、Transfer Engine/RDMA (3)、Store (5)、故障排查 (4 棵诊断树 + 11 个错误速查)、性能调优 (4)。回答第一行固定为「由AI总结，仅供参考！」。

### Q&A 入库

把长文本（学习笔记、issue 讨论、群聊记录）沉淀为经过源码验证的 Q&A 条目：

```bash
/mooncake-agent-skills update-qa "<一大段文本>"
```

**流程**：提取候选 Q&A → 源码验证（完整函数 + 调用链 ±1 层 + 交叉文件）→ 对抗反驳 double check（置信度 ≥ 80 才通过）→ 预览表格用户确认 → 写入 `qa/KNOWLEDGE.md`（附来源注记）。核心原则：**先验证，后入库**。

```bash
# audit 模式：重验已有条目 vs 当前源码，防止条目腐烂（entry rot）
/mooncake-agent-skills update-qa audit          # 审计全库
/mooncake-agent-skills update-qa audit 配置参数  # 只审计指定主题
```

### 清理历史方案

```bash
# 按时间范围清理 proposals（基于 git commit 时间）
/mooncake-agent-skills clear-proposals today     # 今天的方案
/mooncake-agent-skills clear-proposals month     # 本月的方案
/mooncake-agent-skills clear-proposals year      # 今年的方案
```

### PR 代码审查

```bash
# 审查 GitHub PR
/mooncake-agent-skills code-review
/mooncake-agent-skills code-review https://github.com/kvcache-ai/Mooncake/pull/1234
```

### 本地代码审查

```bash
# 全维度审查
/mooncake-agent-skills review

# 指定维度
/mooncake-agent-skills review code errors
/mooncake-agent-skills review simplify
/mooncake-agent-skills review perf-claims   # 性能优化收益真实性（reward-hack 四轴检测）

# 并行加速
/mooncake-agent-skills review all parallel
```

> **⚠️ 平台依赖**：`code-review` 和 `review` 子命令的审查流程**完全依赖 Claude Code 的 Agent 多模型编排能力**（Haiku 资格检查 + Sonnet 并行审查 + 置信度打分）。移植到 ChatGPT Codex / Kimi Code / Z-Code 等其他 AI 编程工具时，需要对应平台的类似能力（多模型并行调用、独立的 confidence scoring agent），否则审查质量会显著下降。

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
| **Code Review** | `code-review/` | 1 | GitHub PR 多 agent 并行审查 + 置信度打分（依赖 Claude Code） |
| **Local Review** | `review/` | 1 | git diff 多维度分析 (comments/tests/errors/types/code/simplify/perf-claims)（依赖 Claude Code） |
| **Quick Q&A** | `qa/` | 1 | 49 配置参数、4 棵诊断树、11 错误速查（回答带 AI 免责声明） |
| **QA 入库** | `update-qa/` | 1 | 文本 → Q&A 提取、源码验证（调用链 ±1 层）、对抗反驳、确认后入库；audit 防腐审计 |
| **Housekeeping** | `clear-proposals/` | 1 | 按 today/month/year 清理历史方案 |

> 子组件各自维护 `SKILL.md`（优化维度 + 领域知识映射）和 `KNOWLEDGE.md`（优化目标知识，人工维护，optimize/plan-feature 的方案内容不回写）。每个 SKILL.md 末尾有维护规则：任何实质性更新需同步检查 `README.md`。

---

## 项目结构

```
MooncakeAgentSkills/
├── SKILL.md                     # /mooncake-agent-skills 入口 (子命令 optimize/plan-feature/code-review/review/qa/update-qa/clear-proposals + git sync)
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
├── code-review/                 # GitHub PR 代码审查
│   └── SKILL.md                 #   5 agent 并行 + 置信度打分 + gh 回帖（依赖 Claude Code）
│
├── review/                      # 本地代码审查
│   └── SKILL.md                 #   git diff 多维度 (comments/tests/errors/types/code/simplify/perf-claims)（依赖 Claude Code）
│
├── clear-proposals/             # 历史方案清理
│   └── SKILL.md                 #   按 today/month/year 清理 proposals
│
├── qa/                          # Mooncake 快速问答
│   ├── SKILL.md                 #   qa 子命令入口（回答带 AI 免责声明）
│   └── KNOWLEDGE.md             #   49 配置参数 + 4 诊断树 + 11 错误速查
│
├── update-qa/                   # 文本整理入库 Q&A
│   └── SKILL.md                 #   提取 Q&A + 源码验证 + 对抗反驳 + 确认后写入 qa/KNOWLEDGE.md；audit 防腐审计
│
├── proposals/                   # 生成的优化方案
└── history/                     # 优化会话日志 + rejected-proposals.md（whiteboard：被否决方案，防重提）
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

### 设计参考：Jitskit(arXiv'26)

Agent 流水线的四处设计借鉴自 Jitskit（LLM agent JIT 系统合成）的实践教训：

| 借鉴点 | 落地位置 | Jitskit 依据 |
|--------|---------|-------------|
| Whiteboard 防重提 | `history/rejected-proposals.md`，Phase 4 查重 + Phase 5 放弃记录 | critic whiteboard：记录已排除设计及证据 |
| 条目防腐审计 | `update-qa audit` 模式 | Auditor：定期审读实现 vs 规范，失效点显式化 |
| 收益真实性审查 | `review perf-claims` 维度 + `code-review` Agent #6 | Reward-Hack Gallery 四轴（规范失配/负载过拟合/装置利用/环境泄漏） |
| Leading indicators + 反作弊验证 | 方案模板「预期收益」「验证方案」section | ablation：leading indicators 收敛差异 1.42-3.75×；不可重构熵/per-run 随机化/状态化验证 |

---

## 维护规则

每个 `SKILL.md` 末尾均有：**任何实质性更新（新增优化维度、修改分析流程、更新路由规则等），必须同步更新本 README.md** — 确保文档始终反映最新能力边界。

---

## ⚠️ 重要：自定义与微调

这个仓库已包含我个人的知识和配置信息（SKILL.md 中的优化维度、KNOWLEDGE.md 中的累积经验、proposals/ 中的历史方案、history/ 中的操作日志）。

如果你要基于此建立自己的 skill：

- **保留所有目录下的 `SKILL.md`** — 它们是子命令的结构骨架（路由规则、领域知识映射、审查流程、子命令分发逻辑）
- **保留 `mooncake/architecture.md`** — 三视角统一模型是优化分析的理论根基，适用于任何 LLM 推理服务平台
- **自由微调 `SKILL.md` 内容** — 添加你的优化维度、修改审查维度、调整置信度阈值、增补 Q&A
- **可以清空 `proposals/`** — 那是我的历史方案，不妨碍功能
- **可以清空 `history/`** — 那是我的操作日志
- **建议保留 `qa/KNOWLEDGE.md`** — 49 个配置参数 + 4 棵诊断树是通用的 Mooncake 知识，与你无关的个人偏好极少
- **必须修改 `config.md`** — 将仓库地址改为你自己的 fork，否则 git push 会推到我的仓库
- **`common/SKILL.md` 建议保留** — 35+ 通用优化模式和 6 维评估框架是跨项目通用的方法论
