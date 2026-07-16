# MooncakeAgentSkills — Claude Code 项目指令

## 项目定位

MooncakeAgentSkills 是一个 **Claude Code Skill**（安装为 `/mooncake-agent-skills`），从领域知识库（domain_knowledge_agent）中检索论文洞察，评估其在真实代码仓库中的可应用性，并生成结构化优化方案。

当前首个对接目标是 **Mooncake**（kvcache-ai/Mooncake），一个面向 LLM 推理的 KVCache 中心化解耦服务平台。

## 架构约定

本项目遵循与 domain_knowledge_agent 一致的文件命名约定：

| 文件 | 类型 | 用途 |
|------|------|------|
| `SKILL.md` | 路由/指令型 | Skill 入口、路由规则、分析流程说明 |
| `KNOWLEDGE.md` | 知识型 | 累积的优化目标知识 (mooncake) 或 Q&A 对 (qa) |
| `CLAUDE.md` | 项目指令型 | 本文件，Claude Code 加载的项目级行为准则 |
| `config.md` | 配置型 | 仓库路径、远程地址等配置 |
| `repo-map.md` | 映射型 | 目标仓库的目录→组件映射表 |
| `architecture.md` | 理论型 | Mooncake 三视角统一模型（分布式存储 + 通信 + Serving）|

## 核心工作流

### 优化分析 (`/mooncake-agent-skills`)

```
/mooncake-agent-skills "<优化问题描述>"
  → git pull (本 repo + Mooncake 源码)
  → 读取 architecture.md (三视角统一模型)
  → 路由到项目目录（mooncake/）
  → 按关键词匹配到组件子目录（两级路由）
  → 读取组件 SKILL.md，获取源代码地图与优化维度
  → 搜索目标仓库源代码
  → 调用 /domain-knowledge 检索相关论文洞察
  → 评估可应用性
  → 生成优化方案到 proposals/
  → 更新 history/optimization-log.md
  → git commit + push
```

### 快速问答 (`/mooncake-qa`)

```
/mooncake-qa "<问题>"
  → qa/SKILL.md: 关键词匹配
  → qa/KNOWLEDGE.md: 检索 Q&A 对
  → 简洁回答（一句话结论 + 必要细节）
  → 未覆盖时提示用户追加到 KNOWLEDGE.md
```

**使用场景**：概念解释、配置问题、故障排查、API 使用。对应 `qa/KNOWLEDGE.md` 中的 30+ 预置 Q&A。

## 与 domain_knowledge_agent 的交互

三种交互方式，按优先级：

1. **直接文件读取（首选）**：当知道目标 KNOWLEDGE.md 路径时，直接 `Read ~/.claude/skills/domain-knowledge/<domain>/<sub>/KNOWLEDGE.md`
2. **Skill 工具调用（语义搜索）**：当需要开放域论文搜索时，使用 `Skill` 工具调用 `domain-knowledge`
3. **交叉引用**：通过 `history/reading-log.md` 追踪论文归档位置，发现相关论文

## 添加新目标项目的规范

当对接新的优化目标（如 vLLM、SGLang）时：

1. 在项目根目录创建 `<project>/` 目录
2. 创建 `<project>/SKILL.md`（路由器，含组件→优化维度→领域知识映射）
3. 创建 `<project>/repo-map.md`（源代码树→组件映射）
4. 为每个组件创建 `<project>/<component>/SKILL.md` + `KNOWLEDGE.md`
5. 在根 `SKILL.md` 添加路由条目
6. 更新 `config.md` 添加新仓库路径配置

## 方案输出规范

所有优化方案必须包含以下结构：

```markdown
# 优化方案: <标题>
- **日期**: YYYY-MM-DD
- **目标组件**: <组件名>
- **引用的领域知识**: <论文列表>

## 问题陈述
## 当前实现分析
## 可应用的论文洞察
## 建议修改
## 预期收益
## 风险与缓解
```

## 行为准则

- **只提方案，不改代码**：本 skill 生成优化建议，不直接修改目标仓库
- **中文为主，术语保留英文**：如 latency、throughput、KV cache
- **引用来源**：每个建议都标注引用的论文和 KNOWLEDGE.md 路径
- **区分事实与推断**：从代码和论文中读到的内容 vs. 基于经验的推断
- **不重复已有方案**：生成方案前先检查 proposals/ 目录，避免重复
- **遵循记忆中的反馈**：代码审查中的发现 ≠ merge blocker，评估时给出诚实的应用建议
