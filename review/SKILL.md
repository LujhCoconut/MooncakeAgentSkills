---
name: mooncake-agent-skills
description: 本地代码审查 — 多维度并行 agent 分析 git diff，结构化输出审查报告
argument-hint: "[all|comments|tests|errors|types|code|simplify|perf-claims|quick]"
---

# review — 本地代码审查

基于 git diff 的多维度代码审查。启动多个专业化 agent 并行分析本地未提交变更，输出结构化的审查报告。

> 入口：`/mooncake-agent-skills review [aspects]`
> 默认审查 `git diff` 的未暂存变更。也可指定审查维度。
> **无参数时**：使用 `AskUserQuestion` 展示维度选项 `[all（默认）| comments | tests | errors | types | code | simplify | perf-claims | quick]` 供用户选择。

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
| `perf-claims` | 收益真实性 | 审查声称的性能优化收益是否真实：靠违反未写明不变量换性能？overfit 特定 benchmark/trace 规律？benchmark 装置不对等？详见下文 perf-claims-reviewer |
| `quick` | 快速扫描 | **minimal 模式**：仅输出 Overall Risk + Top 5 Issues + Missing Tests + context savings，跳过 per-file 展开。适用小改动 / 快速门禁。等效 code-review-graph 的 `detail_level="minimal"` |
| `all` | 全部 | 运行所有适用的维度（默认，不含 quick——quick 是独立输出截断模式，不是额外维度） |

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
- **声称性能优化的变更**（commit message / 注释 / 用户描述中含性能收益声明，或变更明显以提速为目的）：`perf-claims`
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

**聚合前先计算 Overall Risk**：

| 因素 | 权重 | 计算方式 |
|------|------|---------|
| 变更规模 | 基准 | `changed_files` 数量 + `changed_functions` 数量 |
| 影响半径 | 高 | 每个 changed function 的 caller 数量之和（通过 grep/import 追踪估算）；`impacted_files ≥ 3` → +1 风险级 |
| 测试差距 | 高 | `untested_changed_functions / total_changed_functions` 比例；`≥ 0.5` → +1 风险级 |
| 继承/接口变更 | 特例 | 如涉及虚函数/接口签名/跨文件类型修改 → 直接至少 Medium |
| 整体风险 | — | **Low**（≤2 文件且零 caller 影响）\| **Medium**（3-5 文件或 ≤5 callers）\| **High**（>5 文件或 >5 callers 或 untested >50%）|

每个发现的问题额外标注：
- **`dependents: N`** — 该函数/类型被 N 个外部调用者引用（通过 grep 调用方或 import 关系估算），N > 0 时显示

```markdown
# Review Summary

**Overall Risk: Low / Medium / High**（变更 N 文件 · M 函数 · K downstream callers · untested P%）

## Critical Issues (X found) — must fix
- [dimension]: Issue description — file:line — confidence: XX — dependents: N

## Important Issues (X found) — should fix
- [dimension]: Issue description — file:line — confidence: XX — dependents: N

## Suggestions (X found) — nice to have
- [dimension]: Suggestion — file:line

## Missing Tests (Y functions)
- `function_name` in `file` — 建议补: <具体场景>
- （无测试覆盖的 changed function 列表。每个标注建议补什么场景——空值/边界/并发/错误路径/兼容性。借鉴 code-review-graph(arXiv'25) 的 test gap 显式清单——"改了 N 个函数，M 个没测试"比笼统的"测试覆盖不足"可操作得多）

## Strengths
- What's well-done in the changes

## Recommended Action
1. Fix critical issues first
2. Address important issues
3. Review missing test coverage
4. Consider suggestions
5. Re-run review after fixes: /mooncake-agent-skills review

> **上下文消耗**: 本次审查读取 A 个文件 / B 行源码 / 约 C tokens（占全部受影响文件的 D%）。直接读取全部受影响文件预计消耗 E tokens——节省 F%。此统计帮助量化 skill 效率，借鉴 code-review-graph 的 context_savings 自汇报机制。
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

### perf-claims-reviewer（收益真实性审查）

审查性能优化类变更的**收益是否真实**——与其他维度回答"这个问题是真的吗"不同，本维度回答"这个收益是真的吗"。设计依据：Jitskit(arXiv'26) 的 Reward-Hack Gallery——优化压力下，最大的"性能提升"经常来自违反没人写下来的不变量，而非真实优化（其观察到的最极端案例：+37× 吞吐实为 bitmap 假存储，值从未真正落盘）。

**四条作弊轴**（按 Jitskit 分类，Mooncake 语境举例）：

| 轴 | 模式 | 典型手法 | Mooncake 语境示例 |
|----|------|---------|------------------|
| I. 规范失配 | 满足字面规范，违反未写明的不变量 | 原子提交只校验 key 的截断 hash；跨线程绕过所有权直写他人 buffer；same-size 原地更新不推进版本号骗过 seqlock 重试 | 绕过 lease 检查提速读路径；跳过 replica 确认降低写延迟；驱逐时不校验 in-flight 引用 |
| II. 负载过拟合 | 利用测试数据的统计/结构规律，换了负载即错 | 用 key 密度代替 hash（`slot = key & (N-1)`）；值可由 key 推导则只存 seed 重算 | 只对 benchmark 的固定 value_size / 密集 key 空间 / 特定 Zipf 分布成立的优化 |
| III. 装置利用 | 收益来自 benchmark 两侧不对等，而非系统改进 | SUT 与 baseline 的 value clip 不同（+85% 假头条）；只在 THP warmup 窗口内采样 | 对照实验中 RDMA/TCP 配置、MC_BATCH_SIZE、预热状态不一致 |
| IV. 环境泄漏 | 修改共享宿主状态且不复原 | 跑分脚本改 sysctl（如 hugepages）不恢复 | 测试脚本改内核参数/NUMA 绑核影响后续所有测试 |

**审查要点清单**：

1. **归因**：收益能否归因到具体设计改动？commit/注释声称的机制与代码实际做的事一致吗？
2. **反事实**：如果删掉最可疑的捷径（如缓存了某个"恰好"可推导的值），收益还在吗？
3. **泛化**：优化在 key 分布、value 大小、并发度、数据规模变化后**是否仍正确**（不只是仍快）？找出它隐式依赖的负载属性——若该属性是真实部署属性，应要求在注释/文档中显式声明，而非默默依赖
4. **装置对等**：如变更附带 benchmark 数据，检查对照两侧配置逐项相同、采样窗口避开 warmup
5. **验证充分性**：声称的不变量保持（一致性、恢复语义）是否有测试实际行使？"trace 没触发"不等于"保证成立"

**诚实局限（必须遵守）**：Jitskit 报告部分 hack 对代码级审查**不可见**（same-size torn read 被 10 轮代码审计漏掉，触发率 1/2.25 亿读，只能靠端到端正确性 gate 抓到）。因此本维度发现可疑但无法在代码层证实的问题时，**不硬给结论**——输出「需 harness 级验证」建议（如：值注入不可重构熵、per-run 随机 key 置换、重启恢复测试），并按实际置信度打分，不为了凑数拔高。

**perf-claims 专属排除规则**：变更未声称性能收益、也无性能动机时，本维度直接跳过（输出「不适用」），不要把普通功能变更硬套四轴分析。

### quick-mode（快速扫描 / minimal 输出）

与 `all` 不同，`quick` 不是一个新的审查维度，而是一个**输出截断模式**——仍跑 Step 1-2 的维度分析，但 Step 3 聚合时只输出 minimal 子集。借鉴 code-review-graph 的 `detail_level="minimal"`（~100 tokens 摘要 vs ~800+ tokens 完整报告），适用于小改动、快速门禁。

**输出格式**：

```markdown
# Quick Review

**Overall Risk: Low / Medium / High**（N files · M funcs · K callers · untested P%）

## Top Issues (≤5)
1. [dimension] Issue — file:line — conf: XX — dependents: N
...

## Missing Tests (if any)
- `func` in `file` — 建议补: <场景>

## Verdict
✅ Safe to merge / ⚠️ Address N issues first / ❌ Blocked

> **上下文**: 读取 A 文件 / B 行源码 / 约 C tokens（占全部受影响文件的 D%）。完整报告: `/mooncake-agent-skills review all`
```

**规则**：
- Top Issues 取 Critical + Important 中置信度最高的 5 条
- 所有 Critical（≥90）必须出现在 Top Issues 中，哪怕超过 5 条
- Missing Tests 无内容时可省略
- Verdict 依据：无 Critical + Overall Risk Low → Safe；有 Important 或 Risk Medium → ⚠️；有 Critical 或 Risk High → ❌

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

# 审查性能优化 PR 的收益真实性
/mooncake-agent-skills review perf-claims

# 快速门禁（minimal 输出，小改动适用）
/mooncake-agent-skills review quick

# 并行审查加速
/mooncake-agent-skills review all parallel
```

## 维护规则

当本 SKILL.md 有实质性更新，必须同步检查并更新根 `README.md` 中的能力表和项目结构。
