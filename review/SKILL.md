---
name: mooncake-agent-skills
description: 本地代码审查 — 多维度并行 agent 分析 git diff，结构化输出审查报告
---

# review — 本地代码审查

基于 git diff 的多维度代码审查。启动多个专业化 agent 并行分析本地未提交变更，输出结构化的审查报告。

> 入口：`/mooncake-agent-skills review [aspects]`
> 默认审查 `git diff` 的未暂存变更。也可指定审查维度。
> **无参数时**：使用 `AskUserQuestion` 展示维度选项 `[all（默认）| comments | tests | errors | types | code | simplify]` 供用户选择。

## 审查维度

用户可通过参数指定维度，默认走 `all`：

| 维度 | 关键词 | Agent 职责 |
|------|--------|-----------|
| `comments` | 注释 | 验证代码注释准确性 vs 实际代码；识别注释腐烂（comment rot）；检查文档完整性 |
| `tests` | 测试 | 审查行为测试覆盖率；识别关键缺口；评估测试质量 |
| `errors` | 错误处理 | 寻找静默失败（silent failures）；审查 catch 块；检查错误日志 |
| `types` | 类型 | 分析类型封装性；审查 invariant 表达；评级类型设计质量 |
| `code` | 通用 | CLAUDE.md 合规检查；Bug 检测；通用代码质量审查 |
| `simplify` | 简化 | 简化复杂代码；改善可读性；应用项目标准；保持功能不变 |
| `all` | 全部 | 运行所有适用的维度（默认） |

### 审查上下文要求

**每个 agent 在分析时必须扩展上下文，不可只看 diff 行本身**：

1. **读取完整函数**：对每个变更点，读取变更所在函数的完整代码（从函数签名到结尾），不允许仅看 diff 片段
2. **追踪调用链**：识别变更函数/变量的调用者和被调用者，至少向上追踪 1 层（谁调用了这个函数）和向下追踪 1 层（这个函数调用了谁）
3. **交叉文件关联**：如果变更涉及多个文件（如修改了一处接口签名），必须读取所有受影响文件的上下文
4. **上下文加权**：发现的问题必须标注是从多大的上下文中得出的——`[diff-only]` 表示仅看 diff 可发现，`[func-context]` 表示读完整函数后发现，`[call-chain]` 表示追踪调用链后发现

## 审查流程

### Step 1: 确定范围

```bash
git diff --name-only          # 变更文件列表
git diff --stat               # 变更统计
```

判断每个文件的类型，决定哪些审查维度适用：
- **始终适用**：`code`（通用审查）
- **测试文件变更**：`tests`
- **注释/文档新增**：`comments`
- **错误处理变更**：`errors`
- **类型/接口新增或修改**：`types`
- **在通过审查后**：`simplify`

### Step 2: 启动审查 Agent

**默认串行**（一个维度一个维度来，便于交互理解），用户可指定 `parallel` 并行：
```
/mooncake-agent-skills review all parallel
```

每个 agent 使用 **Opus 模型**，输出该维度的详细发现。所有 agent 遵循：
- 仅报告置信度 ≥ 80 的问题
- 区分新引入的问题 vs 已有问题
- 每个问题附带：文件路径 + 行号 + 置信度 + 修复建议

### Step 3: 聚合结果

```markdown
# Review Summary

## Critical Issues (X found) — must fix
- [dimension]: Issue description — file:line — confidence: XX

## Important Issues (X found) — should fix
- [dimension]: Issue description — file:line — confidence: XX

## Suggestions (X found) — nice to have
- [dimension]: Suggestion — file:line

## Strengths
- What's well-done in the changes

## Recommended Action
1. Fix critical issues first
2. Address important issues
3. Consider suggestions
4. Re-run review after fixes: /mooncake-agent-skills review
```

### Step 4: 严重问题 Double Check

**对聚合结果中所有 Critical（置信度 ≥ 90）的问题，必须进行二次验证（double check）**：

1. **启动独立的 Haiku agent**，专门尝试**反驳**该问题：
   - 是否存在合理的代码路径使得该问题不会触发？
   - 是否有上层调用者已经做了防护，使得该问题在实践中不可能出现？
   - 是否有其他文件中的逻辑（未在 diff 中）覆盖了该边界情况？
2. **读取更多上下文**：double check agent 必须读取比原始 agent 更广的上下文——包括调用者的调用者、被调用的所有子函数、相关的头文件/类型定义
3. **输出裁决**：
   - **确认（confirmed）** — 反驳失败，问题确实存在
   - **降级（downgraded）** — 问题存在但有额外防护，降为 Important (80-89)
   - **否定（rejected）** — 误报，从报告中移除，说明反驳理由
4. 仅在 double check 确认后，才将问题列为最终报告中的 Critical

**未通过 double check 的问题不进入最终报告**。如果所有 Critical 都被否决/降级，诚实报告——不凑数。

## 审查 Agent 详细说明

### code-reviewer（通用审查）

核心 agent，执行本地代码审查：

**审查范围**：默认审查 `git diff` 的未暂存变更。用户可指定特定文件。

**CLAUDE.md 合规**：
- 验证 import 模式、框架约定、语言特定风格
- 函数声明、错误处理、日志记录、测试实践
- 平台兼容性、命名约定

**Bug 检测**：
- 逻辑错误、null/undefined 处理
- 竞态条件、内存泄漏
- 安全漏洞、性能问题

**代码质量**（聚焦重大问题时）：
- 代码重复、缺失关键错误处理
- 可访问性问题、测试覆盖不足

**置信度标准**：

| 范围 | 含义 |
|------|------|
| 0-25 | 大概率误报或已有问题 |
| 26-50 | 轻微 nitpick，未被 CLAUDE.md 明确要求 |
| 51-75 | 有效但低影响 |
| 76-90 | 重要问题需处理 |
| 91-100 | 严重 bug 或明确的 CLAUDE.md 违规 |

**仅报告 ≥ 80 分的问题**。如无高置信度问题，确认代码符合标准并摘要通过。

输出按严重度分组：Critical (90-100) 和 Important (80-89)。

## 排除规则

以下情况**不报告**：
- 已有问题（git blame 确认非本 diff 引入）
- 看起来像 bug 但验证后实际不是
- Linter / typechecker / 编译器会捕获的（假定 CI 会跑）
- 通用代码质量（除非 CLAUDE.md 明确要求）
- 被 lint ignore 明确 silenced 的

## 使用示例

```bash
# 默认全维度审查
/mooncake-agent-skills review

# 指定维度
/mooncake-agent-skills review code errors

# 仅简化
/mooncake-agent-skills review simplify

# 并行审查加速
/mooncake-agent-skills review all parallel
```

## 维护规则

当本 SKILL.md 有实质性更新，必须同步检查并更新根 `README.md` 中的能力表和项目结构。
