# MooncakeAgentSkills — Claude Code 项目指令

## 项目定位

MooncakeAgentSkills 是一个 **Claude Code Skill**（安装为 `/mooncake-agent-skills`），提供七个子命令：`optimize`（源码级优化分析）、`plan-feature`（新功能设计方案）、`code-review`（GitHub PR 审查）、`review`（本地代码审查）、`qa`（快速问答）、`update-qa`（文本整理入库 Q&A）、`clear-proposals`（清理历史方案）。与 domain_knowledge_agent 联动实现论文洞察驱动的优化方案与设计方案生成。

当前首个对接目标是 **Mooncake**（kvcache-ai/Mooncake），一个面向 LLM 推理的 KVCache 中心化解耦服务平台。

## 架构约定

本项目遵循与 domain_knowledge_agent 一致的文件命名约定：

| 文件 | 类型 | 用途 |
|------|------|------|
| `SKILL.md` | 路由/指令型 | Skill 入口、路由规则、分析流程说明 |
| `KNOWLEDGE.md` | 知识型 | 优化目标知识 (mooncake，人工维护) 或 Q&A 对 (qa) |
| `CLAUDE.md` | 项目指令型 | 本文件，Claude Code 加载的项目级行为准则 |
| `config.md` | 配置型 | 仓库路径、远程地址等配置 |
| `repo-map.md` | 映射型 | 目标仓库的目录→组件映射表 |
| `architecture.md` | 理论型 | Mooncake 三视角统一模型（分布式存储 + 通信 + Serving）|

## 核心工作流

### 优化分析 (`optimize` 子命令)

```
/mooncake-agent-skills optimize "<优化问题描述>"
  → git pull (本 repo + Mooncake 源码)
  → 读取 architecture.md (三视角统一模型)
  → 路由到项目目录（mooncake/）
  → 按关键词匹配到组件子目录（两级路由）
  → 读取组件 SKILL.md，获取源代码地图与优化维度
  → 搜索目标仓库源代码
  → 调用 /domain-knowledge 检索相关论文洞察
  → Phase 4.5: 审查已有代码（完整函数 + 调用链 ±1 层，避免忽略已实现功能）
  → 评估可应用性
  → Phase 5: 展示预览表格，用户确认
  → Phase 6: 生成优化方案到 proposals/optimize/
  → 更新 history/optimization-log.md
  → git commit + push
```

### 功能设计 (`plan-feature` 子命令)

```
/mooncake-agent-skills plan-feature "<功能需求描述>"
  → git pull (本 repo + Mooncake 源码)
  → 读取 architecture.md (三视角统一模型)
  → 路由到项目目录（mooncake/）
  → Phase 2: 基础设施复用分析（搜索已有接口/模式/类似实现）
  → Phase 3: 设计模式检索（领域知识库）
  → Phase 4: 方案设计（复用清单 + 新增代码 + 独立程度判断）
  → Phase 4.5: 已有代码审查（避免设计已有功能）
  → Phase 5: 展示预览表格，用户确认
  → Phase 6: 生成设计方案到 proposals/feature/
  → 更新 history/optimization-log.md
  → git commit + push
```

### 快速问答 (`qa` 子命令)

```
/mooncake-agent-skills qa "<问题>"
  → qa/SKILL.md: 关键词匹配
  → qa/KNOWLEDGE.md: 检索 Q&A 对
  → 简洁回答（一句话结论 + 必要细节）
  → 未覆盖时提示用户追加到 KNOWLEDGE.md
```

**使用场景**：概念解释、配置问题、故障排查、API 使用。对应 `qa/KNOWLEDGE.md` 中的 30+ 预置 Q&A。回答第一行必须是「由AI总结，仅供参考！」。

### 文本整理入库 (`update-qa` 子命令)

```
/mooncake-agent-skills update-qa "<一大段文本>"
  → Step 1: 提取候选 Q&A（问题 + 答案 + 目标主题 + 可验证声明清单，与已有条目查重）
  → Step 2: 源码验证（完整函数 + 调用链 ±1 层 + 交叉文件，标注证据层级）
  → Step 3: 对抗反驳 double check（confirmed / corrected / rejected，置信度 ≥ 80 才通过）
  → Step 4: 展示预览表格（含 rejected 条目及反驳理由），用户确认
  → Step 5: 写入 qa/KNOWLEDGE.md（附来源注记）
  → git commit + push
```

**使用场景**：把学习笔记、issue 讨论、群聊记录等长文本沉淀为经过源码验证的 Q&A 条目。核心原则：**先验证，后入库**——未经验证的内容不允许写入。

### PR 审查 (`code-review` 子命令)

```
/mooncake-agent-skills code-review [PR URL 或 PR 号]
  → 资格检查 (跳过 closed/draft/已审查 PR)
  → 收集 CLAUDE.md 上下文
  → 5 个并行 agent 独立审查 (CLAUDE.md 合规 ×2, bug 扫描, git blame 历史, 过往 PR 交叉引用)
  → 置信度打分 (0-100)
  → Step 4.5: Critical 问题 double check (反驳式验证)
  → 过滤 ≥ 80 分
  → gh CLI 回帖
```

**使用场景**：GitHub PR 自动化审查。需要 `gh` CLI 已认证。

### 本地审查 (`review` 子命令)

```
/mooncake-agent-skills review [comments|tests|errors|types|code|simplify|all]
  → 确定 git diff 范围
  → 按维度启动专业化 agent (串行默认, parallel 并行)
  → 每个 agent 必须: 读取完整函数 + 追踪调用链 ±1 层 + 交叉文件关联
  → Step 4: Critical 问题 double check (反驳式验证)
  → 聚合结果: Critical / Important / Suggestions / Strengths
  → 仅报告置信度 ≥ 80 的问题
```

**使用场景**：提交前本地代码审查、pre-PR 检查。

### 清理历史方案 (`clear-proposals` 子命令)

```
/mooncake-agent-skills clear-proposals today|month|year
  → git log 查找时间范围内的 proposal 文件（proposals/optimize/ + proposals/feature/）
  → 展示摘要表格
  → 用户确认后 git rm + commit + push
```

**使用场景**：定期清理过期的优化方案和设计方案，避免 proposals/ 目录膨胀。

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

- **只提方案，不改代码**：本 skill 生成优化/设计建议，不直接修改目标仓库
- **中文为主，术语保留英文**：如 latency、throughput、KV cache
- **引用来源**：每个建议都标注引用的论文和 KNOWLEDGE.md 路径
- **区分事实与推断**：从代码和论文中读到的内容 vs. 基于经验的推断
- **不重复已有方案**：生成方案前先检查 `proposals/optimize/` 和 `proposals/feature/` 目录，避免重复
- **proposal 不回写 KNOWLEDGE.md**：optimize / plan-feature 的方案内容只写入 `proposals/` 和 `history/optimization-log.md`；组件 `KNOWLEDGE.md` 由人工维护，本 skill 对其只读不写（`qa/KNOWLEDGE.md` 例外：用户主动追加，或 `update-qa` 经源码验证 + 用户确认后写入）
- **qa 回答带免责声明**：`qa` 子命令的回答第一行必须是「由AI总结，仅供参考！」
- **先预览再写入**：optimize 和 plan-feature 在生成完整方案前必须先展示预览表格，用户确认后才写入文件
- **审查已有代码**：Phase 4.5 必须审查相关已有代码（完整函数 + 调用链），避免优化建议/设计方案忽略已实现功能
- **遵循记忆中的反馈**：代码审查中的发现 ≠ merge blocker，评估时给出诚实的应用建议

## 代码审查行为准则（`code-review` / `review` 子命令）

- **对抗验证**：每个发现的问题都需要独立的置信度打分 agent 来尝试**反驳**——只有反驳不成立的问题才进入最终报告
- **仅报告高置信度**：默认阈值 80（0-100 评分），低于阈值的问题不报告
- **误报排除**：明确列出不报告的场景（已有问题、linter 可捕获、lint ignore 明确 silenced、非本 diff 引入等）
- **区分严重度**：Critical (90-100) 和 Important (80-89)，报告中分别展示
- **不修改代码**：审查只提问题，不直接修改。这与 optimize 子命令的「只提方案，不改代码」原则一致
- **诚实评估合并状态**：即使发现多个低分问题，如果无一达到阈值，也应诚实地输出"No issues found"而非凑数
