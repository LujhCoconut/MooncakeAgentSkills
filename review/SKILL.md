---
name: mooncake-agent-skills
description: 本地代码审查 — 多维度并行 agent 分析 git diff，结构化输出审查报告
---

# review — 本地代码审查

基于 git diff 的多维度代码审查。启动多个专业化 agent 并行分析本地未提交变更，输出结构化的审查报告。

> 入口：`/mooncake-agent-skills review [aspects]`
> 默认审查 `git diff` 的未暂存变更。也可指定审查维度。

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
