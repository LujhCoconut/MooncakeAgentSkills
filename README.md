# MooncakeAgentSkills

> 🧠 基于领域知识的代码优化分析 skill — 从 103+ 篇论文洞察中寻找落地机会

[![Skill](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://claude.ai/code)
[![Target](https://img.shields.io/badge/Target-Mooncake-orange)](https://github.com/kvcache-ai/Mooncake)
[![License](https://img.shields.io/badge/License-MIT-green)](./LICENSE)

MooncakeAgentSkills 是一个 **Claude Code Skill**，提供 `/advanced-optimize` 命令。它与 [domain_knowledge_agent](https://github.com/LujhCoconut/domain_knowledge_agent) 联动，自动扫描代码仓库、检索论文洞察、评估可应用性并生成结构化优化方案。

首个对接目标是 **[Mooncake](https://github.com/kvcache-ai/Mooncake)** — 由 Moonshot AI 开源的 LLM 推理服务平台（FAST 2025 Best Paper），采用 KVCache 中心化的 Prefill-Decode 分离架构。

---

## 工作原理

```
/advanced-optimize Mooncake "降低 RDMA 传输延迟"
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│  Phase 1: 路由解析                                            │
│  mooncake/SKILL.md 关键词匹配 → transfer-engine 组件           │
├──────────────────────────────────────────────────────────────┤
│  Phase 2: 源码分析                                            │
│  repo-map.md 定位代码 → 搜索/阅读关键源文件 → 理解当前实现       │
├──────────────────────────────────────────────────────────────┤
│  Phase 3: 领域知识检索                                         │
│  直接读取 domain_knowledge_agent/KNOWLEDGE.md                  │
│  + 调用 /domain-knowledge 语义搜索                              │
│  → 找到 TMO(OSDI'26) 多路径调度 / RDMA 拥塞控制等洞察            │
├──────────────────────────────────────────────────────────────┤
│  Phase 4: 可应用性评估                                         │
│  common/SKILL.md 评估框架: 技术可行性 × 收益 × 风险 × 实现代价    │
│  → 评级: 可直接应用 / 需适配 / 仅参考 / 不适用                    │
├──────────────────────────────────────────────────────────────┤
│  Phase 5: 方案生成                                             │
│  结构化优化提案 → proposals/                                    │
│  更新 KNOWLEDGE.md → git commit + push                        │
└──────────────────────────────────────────────────────────────┘
```

## 安装

```bash
# 1. 克隆到 Claude Code skills 目录
git clone git@github.com:LujhCoconut/MooncakeAgentSkills.git ~/.claude/skills/mooncake-agent-skills/

# 2. 确保前置依赖已安装
git clone git@github.com:LujhCoconut/domain_knowledge_agent.git ~/.claude/skills/domain-knowledge/
git clone https://github.com/kvcache-ai/Mooncake.git ~/src/Mooncake
```

> 💡 可通过环境变量 `MOONCAKE_REPO` 自定义 Mooncake 源码路径，默认 `~/src/Mooncake`。

## 使用示例

### 传输引擎优化
```bash
/advanced-optimize Mooncake "降低 RDMA 传输延迟"
/advanced-optimize Mooncake "TENT 多路径切片调度如何自适应链路质量"
/advanced-optimize Mooncake "GPU-NIC 拓扑发现是否可用于调度优化"
```

### 分布式存储优化
```bash
/advanced-optimize Mooncake "改进 KV cache 驱逐策略，引入访问频率感知"
/advanced-optimize Mooncake "Master 元数据服务如何实现高可用"
/advanced-optimize Mooncake "G1→G2→G3 三级存储的迁移策略优化"
```

### 缓存路由优化
```bash
/advanced-optimize Mooncake "Conductor 前缀缓存索引结构优化"
/advanced-optimize Mooncake "XXH3-64 哈希在亿级 key 下的碰撞风险"
```

### 框架集成优化
```bash
/advanced-optimize Mooncake "vLLM Connector 的 Python 绑定零拷贝路径"
/advanced-optimize Mooncake "SGLang HiCache L3 后端延迟优化"
```

### 构建 & 运维
```bash
/advanced-optimize Mooncake "CMake 构建并行化与增量编译"
/advanced-optimize Mooncake "端到端可观测性：metrics + tracing + logging"
/advanced-optimize Mooncake "RDMA 链路故障的快速检测和透明降级"
```

## Mooncake 组件覆盖

| 组件 | 目录 | 子组件数 | 覆盖要点 |
|------|------|:--------:|---------|
| **Transfer Engine / TENT** | `mooncake/transfer-engine/` | 4 | transport (RDMA/TCP/NVLink), tent (切片调度/遥测), memory (注册/分配器), topology (GPU-NIC/NUMA) |
| **Mooncake Store** | `mooncake/store/` | 4 | storage-backend (DRAM/SSD/三级存储), master (HA/租约/驱逐), client (读写/连接池/P2P), replication (副本/亲和性/拓扑) |
| **Conductor** | `mooncake/conductor/` | 1 | 哈希索引、前缀匹配、事件管线、多租户 |
| **Framework Integrations** | `mooncake/integrations/` | 1 | pybind11 零拷贝、vLLM/SGLang 连接器 |
| **Build & Deploy** | `mooncake/build-deploy/` | 1 | CMake 并行编译、CI/CD、Wheel 打包 |
| **Operations & SRE** | `mooncake/operations/` | 1 | 可观测性、故障恢复、基准测试 |

> 子组件各自维护 `SKILL.md`（优化维度 + 领域知识映射）和 `KNOWLEDGE.md`（累积优化目标）。

## 项目结构

```
MooncakeAgentSkills/
├── SKILL.md                     # /advanced-optimize 入口 (路由 + git sync)
├── CLAUDE.md                    # 项目指令 (行为准则、扩展规范)
├── config.md                    # 仓库路径、远程地址、环境变量
├── README.md                    # 本文件
├── common/                      # 跨项目通用优化方法论
│   └── SKILL.md                 #   优化模式目录 (35+ 模式)、6 步分析流程、
│                               #   6 维评估框架、反模式警示
├── mooncake/                    # Mooncake 专项技能
│   ├── architecture.md          #   **三视角统一模型** (分布式存储 + 高性能通信 + LLM Serving)
│   ├── SKILL.md                 #   组件路由器 (两级路由)
│   ├── repo-map.md              #   源码树 → 子组件映射 (含子组件标注)
│   ├── transfer-engine/         #   Transfer Engine / TENT
│   │   ├── SKILL.md             #     子组件路由器
│   │   ├── transport/           #     传输协议 (RDMA/TCP/NVLink/EFA/CXL)
│   │   ├── tent/                #     TENT 运行时 (切片/选择/遥测)
│   │   ├── memory/              #     内存管理 (注册/分配/零拷贝)
│   │   └── topology/            #     拓扑发现 (GPU-NIC/NUMA)
│   ├── store/                   #   Mooncake Store
│   │   ├── SKILL.md             #     子组件路由器
│   │   ├── storage-backend/     #     存储后端 (DRAM/SSD/G1-G2-G3)
│   │   ├── master/              #     元数据主控 (HA/租约/驱逐)
│   │   ├── client/              #     客户端数据路径 (读写/连接/P2P)
│   │   └── replication/         #     副本放置 (亲和性/拓扑感知)
│   ├── conductor/               #   Conductor KV Indexer
│   ├── conductor/               #   Conductor 优化
│   ├── integrations/            #   框架集成优化
│   ├── build-deploy/            #   构建部署优化
│   └── operations/              #   运维 SRE 优化
├── proposals/                   # 生成的优化方案 (.md)
└── history/                     # 优化会话日志
```

## 与 domain_knowledge_agent 的联动

```
domain_knowledge_agent                  MooncakeAgentSkills
┌────────────────────────┐             ┌──────────────────────────┐
│ 103 篇论文              │             │ 6 个组件                    │
│ 7 个领域                │  论文洞察    │ 35 个优化维度               │
│                        │  ────────→  │                           │
│ SKILL.md (路由)         │             │ SKILL.md (路由+分析)        │
│ KNOWLEDGE.md (知识)     │  ←─ 新知 ─  │ KNOWLEDGE.md (优化目标)     │
│                        │   反馈       │                           │
│ 论文归档 + 阅读笔记      │             │ 源码分析 + 方案生成          │
└────────────────────────┘             └──────────────────────────┘
```

**三种交互方式**：
1. **直接文件读取** — 已知路径时直接 Read `~/.claude/skills/domain-knowledge/<domain>/KNOWLEDGE.md`
2. **Skill 工具调用** — 开放域搜索时 invoke `/domain-knowledge` 语义检索
3. **交叉引用** — 通过 `history/reading-log.md` 发现关联论文

**一致性约定**：两个项目共享相同的文件命名规则（`SKILL.md` = 路由/指令，`KNOWLEDGE.md` = 知识累积）、Git 同步策略（pre-op pull + post-op push）和知识归档格式。

## 通用优化方法论

`common/SKILL.md` 定义了跨语言、跨项目的优化分析框架：

| 类别 | 覆盖模式 |
|------|---------|
| **数据通路** | 零拷贝、批量处理、预取、流水线、异步化 |
| **资源调度** | 负载均衡、动态调度、亲和性调度、优先级、多路径 |
| **存储与缓存** | 分级存储、驱逐策略、写时复制、压缩编码、分片 |
| **并发与并行** | 锁粒度、无锁结构、RCU、工作窃取 |
| **可靠性** | 故障检测、优雅降级、重试退避、熔断、数据校验 |

每个优化建议都经过 **6 维评估**（技术可行性、架构兼容性、性能收益、实现代价、风险等级、时机合适）后给出 **4 级评级**。

## 后续计划

- [ ] 支持 vLLM、SGLang、TensorRT-LLM 等更多推理框架
- [ ] 方案效果追踪（proposal → implementation → benchmark）
- [ ] 自动发现优化机会（基于代码 pattern 扫描）
- [ ] 与 Mooncake 上游保持同步更新 repo-map.md

## License

MIT © LujhCoconut
