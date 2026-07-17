---
name: mooncake-agent-skills
description: GitHub PR 自动化代码审查 — 多 agent 并行 + 置信度打分 + 对抗验证
argument-hint: "<PR-URL 或 PR 号>"
---

# code-review — GitHub PR 自动化审查

对 GitHub Pull Request 进行多 agent 并行 + 置信度打分的自动化代码审查，最终以结构化格式回帖到 PR。

> 入口：`/mooncake-agent-skills code-review [PR URL 或 PR 号]`
> 如果省略 PR 参数，自动从当前分支推导关联 PR。

## 审查流程

### Step 1: 资格检查

用 Haiku agent 检查 PR 是否满足审查条件。以下情况跳过：
- PR 已关闭 (closed)
- PR 是草稿 (draft)
- 不需要审查的自动化 PR（如 dependabot）
- 已经由本方审查过（检查已有 review comments）

### Step 2: 收集上下文

用 Haiku agent 收集相关 CLAUDE.md 文件：
- 仓库根目录的 CLAUDE.md（如果存在）
- PR 修改文件所在目录的 CLAUDE.md
- 返回文件路径列表（不返回内容）

用另一个 Haiku agent 提取 PR 摘要：
- 变更概述
- 修改的文件列表
- 关键改动点

### Step 3: 多 Agent 并行审查

启动 **5 个并行 Sonnet agent**，各从不同维度独立审查；**当 PR 声称性能优化时**（标题/描述/commit message 含性能收益声明），额外启动第 6 个 agent：

| Agent | 视角 | 审查内容 |
|-------|------|---------|
| #1 | CLAUDE.md 合规 | 检查变更是否符合 CLAUDE.md 规范（注意：CLAUDE.md 是 Claude 写代码的指引，并非所有条目都适用于审查） |
| #2 | 浅层 Bug 扫描 | 仅看 diff 本身，快速扫描明显 bug。聚焦大 bug，忽略小问题、nitpick、明显误报。避免读取 diff 外的上下文 |
| #3 | 历史上下文 | 读取 git blame 和修改代码的历史，从历史演进角度发现潜在的上下文相关 bug |
| #4 | 过往 PR 交叉引用 | 检查之前 touch 过这些文件的 PR，查看是否有仍然适用于当前 PR 的 review comments |
| #5 | 代码注释一致性 | 读取修改文件中的代码注释，确保 PR 的变更与注释中的指引一致 |
| #6 | 收益真实性（条件启用） | 按 `review/SKILL.md` 的 perf-claims-reviewer 执行：四条作弊轴（规范失配/负载过拟合/装置利用/环境泄漏）+ 审查要点清单 + 诚实局限（代码层无法证实的输出「需 harness 级验证」而非硬下结论） |

#### 审查上下文要求

**所有 agent 在分析时必须扩展上下文，不可只看 diff 行本身**：

1. **读取完整函数**：对每个变更点，读取变更所在函数的完整代码（从函数签名到结尾），不允许仅看 diff 片段
2. **追踪调用链**：识别变更函数/变量的调用者和被调用者，至少向上追踪 1 层（谁调用了这个函数）和向下追踪 1 层（这个函数调用了谁）
3. **交叉文件关联**：如果变更涉及多个文件（如修改了一处接口签名），必须读取所有受影响文件的上下文
4. **上下文标注**：每个发现必须注明是从多大的上下文中得出的——`[diff-only]` `[func-context]` `[call-chain]`

每个 agent 返回：问题列表 + 每个问题的原因说明（如 CLAUDE.md 违规、bug、历史上下文等）+ 上下文标注。

### Step 4: 置信度打分

对 Step 3 发现的每个问题，启动**并行 Haiku agent** 进行独立置信度打分（0-100）：

| 分数 | 含义 |
|------|------|
| 0 | 不成立 — 明显误报，或已有问题 |
| 25 | 低置信度 — 可能是真的，但也可能是误报。若是风格问题，未被 CLAUDE.md 明确要求 |
| 50 | 中等 — 验证为真问题，但可能是 nitpick 或实践中不常触发 |
| 75 | 高置信度 — 确认是真问题，会在实践中触发。很重要，直接影响功能，或被 CLAUDE.md 明确提及 |
| 90-100 | 绝对确定 — 确认会高频触发，证据直接支持 |

**对 CLAUDE.md 相关的 issue**：打分 agent 必须二次确认 CLAUDE.md 确实明确提及了该问题。

### Step 4.5: 高严重度问题 Double Check

**对置信度 ≥ 90 的问题，在发布前必须进行二次验证（double check）**：

1. **启动独立的 Haiku agent**，专门尝试**反驳**该问题：
   - 是否存在合理的代码路径使得该问题不会触发？
   - 是否有上层调用者已经做了防护，使得该问题在实践中不可能出现？
   - 是否有其他文件中的逻辑（未在 diff 中）覆盖了该边界情况？
   - 在更大的代码上下文（调用链、所有调用者的调用者）中该问题是否仍然成立？
2. **读取更广的上下文**：double check agent 必须读取比原始审查 agent 更广的上下文——包括变更函数的调用者、被调用函数、以及相关的类型/接口定义
3. **输出裁决**：
   - **confirmed** — 反驳失败，问题确实存在，保留原始分数
   - **downgraded** — 问题存在但有额外防护，降为 Important (75-89)，需说明降级理由
   - **rejected** — 误报，从最终报告中移除，需说明反驳理由
4. 仅 **confirmed** 和 **downgraded** 的问题进入 Step 5 发布

### Step 5: 过滤与发布

- 过滤：仅保留 ≥ 80 分的问题
- 如果无高置信度问题 → 不发布评论，退出
- 用 Haiku agent 再次执行资格检查（PR 状态可能在审查过程中变化）

用 `gh` CLI 将结果回帖到 PR，格式如下：

```
## Code review

Found N issues:

1. <问题简述> (CLAUDE.md says "<相关条文>")

<link: 完整 sha + 文件路径 + 行号范围, 如 https://github.com/owner/repo/blob/<full-sha>/path#L10-L15>

2. <问题简述> (bug due to <文件和代码上下文>)

<link>

🤖 Generated with [Claude Code](https://claude.ai/code)
```

如无问题：

```
## Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

🤖 Generated with [Claude Code](https://claude.ai/code)
```

## 误报排除指南

以下情况**不报告**：

- 已有问题 (pre-existing)，非 PR 引入
- 看起来像 bug 但实际不是
- 资深工程师不会指出的 pedantic nitpick
- Linter / typechecker / 编译器会捕获的问题（无需手动跑这些工具）
- 通用代码质量问题（缺乏测试覆盖、安全审计等），除非 CLAUDE.md 明确要求
- 被 lint ignore comment 明确 silenced 的问题
- 用户未修改行上的问题
- 功能变更可能是故意的，属于 PR 的整体意图

## 链接格式要求

- 必须使用完整 git sha（不可缩写）
- 格式：`https://github.com/owner/repo/blob/<full-sha>/path#Lstart-Lend`
- 提供至少前后各 1 行上下文
- 仓库名必须匹配审查的仓库

## 依赖

- `gh` CLI 已安装并认证
- 当前目录为 git 仓库，关联 GitHub remote

## 维护规则

当本 SKILL.md 有实质性更新，必须同步检查并更新根 `README.md` 中的能力表和项目结构。
