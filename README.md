# MooncakeAgentSkills

> 基于领域知识的代码优化分析 skill。从论文中提取洞察，评估在真实代码仓库中的可应用性。

## 概述

MooncakeAgentSkills 是一个 Claude Code Skill，提供 `/advanced-optimize` 命令。它与你已有的 [domain_knowledge_agent](https://github.com/LujhCoconut/domain_knowledge_agent) 联动，自动：

1. **解析优化问题** → 定位到目标仓库的具体组件
2. **搜索源代码** → 理解当前实现模式
3. **检索领域知识** → 从 103+ 篇论文中查找可复用洞察
4. **评估可应用性** → 判断洞察是否可落地
5. **生成优化方案** → 结构化方案输出到 `proposals/`

首个对接目标是 **[Mooncake](https://github.com/kvcache-ai/Mooncake)**，一个面向 LLM 推理的 KVCache 中心化解耦服务平台（FAST 2025 Best Paper）。

## 安装

```bash
# 克隆到 Claude Code skills 目录
git clone git@github.com:LujhCoconut/MooncakeAgentSkills.git ~/.claude/skills/mooncake-agent-skills/
```

前置依赖：
- [domain_knowledge_agent](https://github.com/LujhCoconut/domain_knowledge_agent) 已安装到 `~/.claude/skills/domain-knowledge/`
- [Mooncake](https://github.com/kvcache-ai/Mooncake) 源码已克隆到本地（默认 `~/src/Mooncake`，可通过 `MOONCAKE_REPO` 环境变量覆盖）

## 使用

```bash
# 基本用法
/advanced-optimize Mooncake "降低 RDMA 传输延迟"

# 运维优化
/advanced-optimize Mooncake "如何提升 Master 服务的高可用性"

# 网络优化
/advanced-optimize Mooncake "多路径传输调度的拥塞感知优化"

# 存储优化
/advanced-optimize Mooncake "改进 KV cache 的驱逐策略"
```

## 项目结构

```
MooncakeAgentSkills/
├── SKILL.md                     # /advanced-optimize 入口
├── CLAUDE.md                    # 项目指令
├── config.md                    # 仓库路径与同步配置
├── README.md                    # 本文件
├── common/                      # 跨项目通用优化方法论
│   └── SKILL.md                 # 优化模式目录、分析流程、方案模板
├── mooncake/                    # Mooncake 专项优化技能
│   ├── SKILL.md                 # Mooncake 路由器（问题→组件映射）
│   ├── repo-map.md              # 源代码树→组件映射表
│   ├── transfer-engine/         # Transfer Engine / TENT 优化
│   ├── store/                   # Mooncake Store 优化
│   ├── conductor/               # Conductor (KV Indexer) 优化
│   ├── integrations/            # 框架集成优化（vLLM/SGLang/...）
│   ├── build-deploy/            # 构建、部署、CI/CD 优化
│   └── operations/              # 运维、监控、SRE 优化
├── proposals/                   # 生成的优化方案
└── history/                     # 优化会话记录
    ├── SKILL.md
    └── optimization-log.md
```

## 与 domain_knowledge_agent 的关系

```
domain_knowledge_agent                    MooncakeAgentSkills
┌─────────────────────┐                  ┌──────────────────────┐
│ 论文 → 洞察 → 归档   │  ── 检索洞察 ──→  │ 洞察 → 代码评估 → 方案 │
│ KNOWLEDGE.md (知识)  │                  │ KNOWLEDGE.md (目标)   │
│ SKILL.md (路由)      │  ←── 反馈新知 ──  │ SKILL.md (路由)       │
└─────────────────────┘                  └──────────────────────┘
```

- MooncakeAgentSkills **读取** domain_knowledge_agent 的 KNOWLEDGE.md 获取论文洞察
- 优化过程中发现的**新优化模式**可反馈写入 domain_knowledge_agent
- 两者共享相同的文件命名约定（SKILL.md / KNOWLEDGE.md）和 Git 同步策略

## 贡献

本项目当前聚焦 Mooncake。未来计划支持更多目标项目（vLLM、SGLang、TensorRT-LLM 等）。

## 许可

MIT
