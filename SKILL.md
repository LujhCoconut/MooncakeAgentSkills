---
name: mooncake-agent-skills
description: Mooncake 优化分析 · 代码审查 · 快速问答 · 功能设计。子命令 optimize（源码级优化+论文洞察+方案生成）、plan-feature（新功能设计+基础设施复用+方案生成）、code-review（GitHub PR 多 agent 审查）、review（本地 git diff 多维度审查）、qa（概念/配置/故障快速问答）、clear-proposals（按 today/month/year 清理历史方案）。覆盖 Transfer Engine、Store、Conductor、框架集成、构建部署、运维监控。
---

# Mooncake Agent Skills

统一入口，通过子命令分发到不同功能：

- **`optimize`**（默认）— 源码级优化分析：定位组件 → 阅读源码 → 检索论文洞察 → 审查已有代码 → 评估可应用性 → **展示预览表格 → 用户确认** → 生成结构化方案
- **`plan-feature`** — 新功能设计方案：分析现有基础设施可复用性 → 检索知识库设计模式 → 判断独立/集成实现方式 → **展示预览表格 → 用户确认** → 生成设计方案
- **`code-review`** — GitHub PR 自动化审查：多 agent 并行 + 置信度打分 + 对抗验证 → `gh` CLI 回帖
- **`review`** — 本地代码审查：`git diff` 多维度分析 (comments/tests/errors/types/code/simplify) → 结构化报告
- **`qa`** — 快速问答：概念解释、配置查询、故障排查（检索 `qa/KNOWLEDGE.md`）
- **`clear-proposals`** — 清理历史方案：按 today/month/year 删除 proposals，交互确认后执行

## ⚠️ 前置操作：同步远程仓库（必须最先执行）

**在读取任何子目录文件或处理用户请求之前，必须先执行 git pull 同步：**

```bash
cd <本 skill 所在目录> && git pull --rebase
```

- 如果 pull 成功 → 继续处理用户请求
- 如果 pull 因网络问题失败 → 告知用户"⚠️ 无法同步远程仓库，继续使用本地版本"，不阻塞后续操作
- 如果 pull 产生冲突 → 告知用户"⚠️ git pull 产生冲突，请手动解决"，继续处理用户请求不阻塞

## 子命令解析

从用户输入中识别子命令和参数：

| 用户输入模式 | 子命令 | 示例 |
|-------------|--------|------|
| 以 `optimize` 开头，或直接描述优化问题（无显式子命令） | `optimize` | `/mooncake-agent-skills optimize "降低 RDMA 传输延迟"` 或 `/mooncake-agent-skills "改进 KV cache 驱逐策略"` |
| 以 `plan-feature` 开头，或提及「设计」「新增功能」「实现方案」「怎么实现」等 | `plan-feature` | `/mooncake-agent-skills plan-feature "Store 对接云存储需要实现什么"` |
| 以 `code-review` 开头，或提及 PR/审查/回帖 | `code-review` | `/mooncake-agent-skills code-review https://github.com/.../pull/123` |
| 以 `review` 开头，或要求审查本地代码/the diff | `review` | `/mooncake-agent-skills review code errors` |
| 以 `clear-proposals` 开头 | `clear-proposals` | `/mooncake-agent-skills clear-proposals today` |
| 以 `qa` 开头，或问概念/配置/报错类问题 | `qa` | `/mooncake-agent-skills qa "RDMA QP 报错怎么办"` 或 `/mooncake-agent-skills "Mooncake 的 PD 分离是什么意思"` |

**子命令分发逻辑**：
1. 如果 args 以 `optimize`、`plan-feature`、`qa`、`code-review`、`review` 或 `clear-proposals` 开头 → 提取子命令，剩余部分为参数
2. 如果无显式子命令 → 判断参数特征：
   - 包含「优化」「改进」「降低」「提升」「分析代码」「瓶颈」等关键词 → 走 `optimize` 流程
   - 包含「设计」「新增功能」「实现方案」「怎么实现」「架构设计」「功能设计」等关键词 → 走 `plan-feature` 流程
   - 包含「审查」「review」「code review」「PR」「pull request」「diff」等 → 走 `review` 流程（本地）或 `code-review`（如含 PR URL）
   - 包含「是什么」「怎么」「为什么」「报错」「配置」「默认值」「怎么调」等 → 走 `qa` 流程
   - 包含「清理」「删除」「清除」「方案」「proposal」等 → 走 `clear-proposals` 流程
   - 歧义时默认走 `optimize`（宁可深度分析也不少答）
3. `code-review` 子命令：读取 `code-review/SKILL.md` 执行 GitHub PR 审查流程
4. `review` 子命令：读取 `review/SKILL.md` 执行本地多维度代码审查
5. `clear-proposals` 子命令：读取 `clear-proposals/SKILL.md` 按时间范围清理历史方案
6. `plan-feature` 子命令：读取 `plan-feature/SKILL.md` 执行新功能设计分析流程

---

# optimize — 源码级优化分析

基于领域知识的代码优化分析流程。你的任务不是直接凭记忆给优化建议，而是先定位目标仓库的对应组件、阅读源代码、检索相关论文洞察，再评估可应用性并生成结构化方案。

## 参数说明

从用户输入中提取优化问题。由于当前仅对接 Mooncake，无需指定目标项目。

示例：
- `/mooncake-agent-skills optimize "降低 RDMA 传输延迟"`
- `/mooncake-agent-skills optimize "改进 KV cache 驱逐策略"`
- `/mooncake-agent-skills optimize "提升 Master 的 HA"`

## 请求类型判断与路由规则

根据目标项目，路由到对应的项目目录：

| 目标项目 | 判断特征 | 应检索的目录 |
|----------|----------|--------------|
| **Mooncake** | mooncake, kvcache, transfer engine, store, conductor, vLLM connector, SGLang HiCache | `mooncake/` |

如果目标项目暂无对应目录：
1. 告知用户当前仅支持 Mooncake
2. 询问是否要参考 Mooncake 的模式为新项目创建目录结构
3. 说明添加新项目的步骤（参考 CLAUDE.md 中的规范）

## 检索与优化流程

> **注意区分请求类型**：如果用户问的是 Mooncake 的**概念、配置、FAQ 类问题**（如 "Mooncake 怎么安装"、"RDMA QP 报错怎么办"），应该走 `qa` 子命令快速回答，而不是走完整的 optimize 流程。只有当用户明确要求**分析代码并生成优化方案**时才走以下流程。

### Phase 1: 路由到项目

1. 读取 `<project>/SKILL.md`（如 `mooncake/SKILL.md`）
2. 根据优化问题关键词匹配到对应组件
3. 读取 `<project>/repo-map.md` 获取源代码映射

### Phase 2: 组件级分析

4. 读取 `<project>/<component>/SKILL.md`，了解优化维度和领域知识映射
5. 搜索目标仓库的相关源代码（使用 Grep、Glob、Read 等工具）
6. 理解当前实现模式、关键数据结构、算法选择

### Phase 3: 领域知识检索

7. **直接文件读取（首选）**：根据组件 SKILL.md 中预定义的领域知识映射，直接读取 `~/.claude/skills/domain-knowledge/<domain>/<sub>/KNOWLEDGE.md`
8. **Skill 工具调用（语义搜索）**：当需要开放域搜索时，使用 `Skill` 工具调用 `domain-knowledge`：
   ```
   Skill: skill="domain-knowledge", args="<优化问题的中文描述>"
   ```
   例如：`skill="domain-knowledge", args="RDMA 多路径传输调度有哪些优化方案？"`
9. **交叉引用**：读取 `~/.claude/skills/domain-knowledge/history/reading-log.md`，根据「归档位置」列发现相关论文

### Phase 4: 评估与比对

10. 对比论文洞察与当前代码实现
11. 评估每条洞察的可应用性：
    - **可直接应用**：当前实现存在明确 gap，论文方案可直接套用
    - **需适配**：核心思路可迁移，但需要针对 Mooncake 的架构做调整
    - **仅参考**：思路有价值但不完全匹配，提供设计参考
    - **不适用**：明确说明原因
12. 检查 `proposals/optimize/` 目录是否已有相关方案，避免重复
13. 读取 `<project>/<component>/KNOWLEDGE.md`，查看已记录的优化目标

### Phase 4.5: 已有代码审查 — 避免忽略已实现功能

**在生成方案之前，必须对相关的已有代码进行审查式分析**，确认当前代码中是否已经实现了部分优化点：

14. **扩展现有代码搜索范围**：对 Phase 2 中定位的关键源文件，系统性地搜索与优化问题相关的所有函数和类定义
15. **完整函数上下文阅读**：对每个相关函数，读取完整实现（从签名到结尾），不只看 grep 命中的片段
16. **追踪调用链**：识别相关函数的调用者和被调用者，至少向上追踪 1 层（谁调用）和向下追踪 1 层（调用谁）
17. **交叉文件关联**：如果优化点涉及跨文件交互（如 client ↔ master ↔ storage backend），读取所有相关文件的交互路径
18. **标记已有功能**：如果代码中已存在部分实现，在方案的「当前实现分析」中明确标注：
    - ✅ 已实现的部分功能
    - ⚠️ 部分实现但存在 gap 的功能
    - ❌ 未实现的功能
19. 如果发现优化目标的大部分已被实现 → 说明「当前代码已覆盖此优化」，调整方案焦点到剩余 gap

### Phase 5: 展示预览 — 用户确认

**在写入方案文件之前，必须先展示预览表格，由用户决定是否继续：**

20. 向用户展示**方案预览表格**：

```markdown
| 项目 | 内容 |
|------|------|
| **方案标题** | <一句话标题> |
| **目标组件** | <组件名> |
| **可应用性** | 可直接应用 / 需适配 / 仅参考 |
| **已有功能** | ✅ <已实现部分> / ❌ <未实现部分> |
| **P0 建议** | <核心优化点> |
| **P1 建议** | <次要优化点> |
| **引用论文** | <论文 1>, <论文 2>, ... |
| **预估工作量** | <大/中/小>，约 <N> 个文件 |
```

21. 使用 `AskUserQuestion` 询问用户：

```
question: "是否生成完整方案并提交？"
options:
  - "生成完整方案" (Recommended)
  - "调整方案范围"（用户补充说明后再生成）
  - "放弃本次方案"
```

22. **仅在用户选择「生成完整方案」后才写入方案文件**。如果选择「调整方案范围」，根据用户补充说明修改预览后重新确认。如果选择「放弃」，不生成文件。

### Phase 6: 输出方案

23. 按方案模板生成优化方案到 `proposals/optimize/<project>-<component>-<slug>-<date>.md`
24. 更新 `<project>/<component>/KNOWLEDGE.md`（如果发现新的优化目标）
25. 更新 `history/optimization-log.md`
26. 向用户呈现方案摘要

---

# plan-feature — 新功能设计方案

针对尚未实现的大功能，分析现有基础设施的可复用性，结合知识库的设计模式，生成结构化设计方案。与 `optimize` 的区别：`optimize` 关注「现有代码的改进」，`plan-feature` 关注「未实现功能的设计」。

## 参数说明

从用户输入中提取功能需求。由于当前仅对接 Mooncake，无需指定目标项目。

示例：
- `/mooncake-agent-skills plan-feature "Store 对接云存储需要实现什么"`
- `/mooncake-agent-skills plan-feature "支持异构 GPU 节点的 KV cache 分配策略"`

## 设计分析流程

> 与 `optimize` 使用相同的模板和输出规范，但 Phase 2 侧重「基础设施复用分析」而非「现有代码问题」，Phase 4.5 确保不设计已有功能，Phase 5 同样需要用户确认后才生成方案。

### Phase 1: 路由到项目

1. 读取 `<project>/SKILL.md`（如 `mooncake/SKILL.md`）
2. 根据功能需求关键词匹配到对应组件（可能跨多个组件）
3. 读取 `<project>/repo-map.md` 和 `<project>/architecture.md` 获取全貌

### Phase 2: 基础设施复用分析

4. **搜索已有相关代码**：在目标仓库中搜索可能与新功能相关的现有实现，包括：
   - 接口/抽象类定义（如 `FileSystemAdapter`、`StorageBackendInterface`）
   - 设计模式（如 `DistributedStorageBackend` 的 adapter 模式）
   - 配置/工厂类（如 `CreateStorageBackend()` 的后端注册）
   - 类似功能的实现（如 `S3Helper` + `SnapshotObjectStore` → 可复用 S3 SDK 集成）
5. **判断复用方式**：
   - **可直接复用**：现有接口/类可直接作为新功能的构建块，只需新增 adapter/plugin
   - **需扩展接口**：现有接口需要新增虚函数或配置字段
   - **需独立实现**：功能与现有代码无交集，需要独立的代码文件和类层次
6. **读取相关代码的完整上下文**（完整函数 + 调用链 ±1 层），理解现有架构的扩展点

### Phase 3: 设计模式检索（领域知识）

7. 根据功能需求，从知识库中检索相关设计模式和最佳实践
8. 方法与 `optimize` Phase 3 相同：直接文件读取 + Skill 语义搜索 + 交叉引用

### Phase 4: 方案设计

9. 基于基础设施分析 + 领域知识，设计实现方案：
   - **架构定位**：新功能在 Mooncake 三层架构中属于哪一层？与哪些现有组件交互？
   - **复用清单**：列出可直接复用的接口/类/模式
   - **新增代码**：列出需要新增的文件和类，以及它们与现有代码的关系
   - **独立程度**：判断新功能是作为现有组件的扩展（plugin/adapter 模式）还是独立模块

### Phase 4.5: 已有代码审查 — 避免设计已有功能

10. **双重检查**：确认要设计的功能尚未在代码中实现
11. 如果发现代码中已有部分实现 → 调整方案为：在现有实现基础上补充 gap（并标注已实现部分）
12. 方法与 `optimize` Phase 4.5 完全相同

### Phase 5: 展示预览 — 用户确认

13. 与 `optimize` Phase 5 相同，先展示预览表格，用户确认后才生成方案

### Phase 6: 输出方案

14. 按方案模板生成设计方案到 `proposals/feature/<project>-<component>-<slug>-<date>.md`
15. 更新相关 `KNOWLEDGE.md`（如果发现了新的设计模式或优化目标）
16. 更新 `history/optimization-log.md`（标注为 `plan-feature`）
17. 向用户呈现方案摘要

---

# qa — 快速问答

Mooncake 常见问题快速回答入口。你的任务是根据用户问题中的关键词，在 `qa/KNOWLEDGE.md` 中定位相关 Q&A，给出准确、简洁的回答。如果 KNOWLEDGE.md 中没有覆盖，诚实告知并建议用户将问题补充到 Q&A 库。

## 检索流程

1. **关键词匹配**：从用户问题中提取关键词，匹配 `qa/KNOWLEDGE.md` 中的 `##` 二级标题（主题）和 `###` 三级标题（具体问题）
2. **模糊匹配**：如果精确匹配失败，尝试同义词/相关词匹配（如 "RDMA" ↔ "传输"、"TTL" ↔ "租约"）
3. **多主题**：如果问题涉及多个主题，合并回答
4. **未覆盖**：如果 KNOWLEDGE.md 无匹配，回答"当前 Q&A 库未覆盖此问题"，建议用户在 `qa/KNOWLEDGE.md` 中追加

## 回答格式

- **简洁优先**：先给一句话结论，再展开细节
- **引用源码**：如果涉及代码，给出文件路径和关键函数名
- **区分版本**：如果信息可能因 Mooncake 版本而异，说明适用版本
- **关联优化**：如果问题适合走 optimize 深度分析，在回答末尾提示可以用 `optimize` 子命令

---

## 方案输出模板

每个优化方案必须包含以下完整结构：

```markdown
# 优化方案: <一句话标题>

- **日期**: YYYY-MM-DD
- **目标项目**: <项目名>
- **目标组件**: <组件名>
- **优化维度**: <维度名>
- **引用的领域知识**: <论文列表，含 KNOWLEDGE.md 路径>
- **可应用性评级**: 可直接应用 / 需适配 / 仅参考

## 问题陈述
<一段话描述当前存在的问题或优化机会，包含量化数据（如果有）>

## 当前实现分析
<目标仓库中的相关代码路径、关键函数/类、当前采用的算法/策略>
<附上关键代码片段（精简、标注行号）>

## 可应用的论文洞察
<对每条洞察，说明来源论文、核心思路、与当前实现的差距>

### 洞察 1: <洞察标题>（来源：<方案名(会议'年份)>）
- **论文中的做法**: ...
- **当前实现的差距**: ...
- **可迁移性分析**: ...

### 洞察 2: ...

## 建议修改
<具体的、可操作的修改建议，按优先级排列>
1. [P0] ...
2. [P1] ...
3. [P2] ...

## 预期收益
<量化预估，含置信度>
- **延迟**: ...
- **吞吐**: ...
- **资源利用率**: ...
- **可靠性**: ...

## 风险与缓解
| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| ... | 低/中/高 | ... | ... |

## 验证方案
<如何验证优化效果：benchmark、监控指标、A/B 测试等>
```

## 输出规范

- **引用来源**：方案开头说明参考了哪些 SKILL.md、KNOWLEDGE.md 和论文
- **中文为主**，标准术语保留英文（如 latency、throughput、KV cache、RDMA、eviction）
- **区分事实与推断**：从代码/论文中读到的内容是事实；基于经验的判断要明确标注
- **结构化输出**：优先用 bullet、表格、代码块，避免大段散文
- **不确定时说明**：如果领域知识库中没有相关信息，直接说"当前领域知识库没有覆盖这一点"
- **给出诚实的评估**：不是每条论文洞察都适合落地。如果可应用性低，诚实说明，不要强行套用

## 目录索引

- `common/SKILL.md` — 跨项目通用优化方法论、模式目录、评估框架
- `mooncake/SKILL.md` — Mooncake 项目路由（按组件分发）
- `mooncake/architecture.md` — **Mooncake 三视角统一模型**（分布式存储 + 高性能通信 + LLM Serving）
- `mooncake/repo-map.md` — Mooncake 源代码树→组件映射
- `mooncake/transfer-engine/SKILL.md` — Transfer Engine 路由 → transport/tent/memory/topology
- `mooncake/store/SKILL.md` — Store 路由 → storage-backend/master/client/replication
- `mooncake/conductor/SKILL.md` — Conductor (KV Indexer) 优化
- `mooncake/integrations/SKILL.md` — 框架集成优化
- `mooncake/build-deploy/SKILL.md` — 构建与部署优化
- `mooncake/operations/SKILL.md` — 运维与 SRE 优化
- `mooncake/queueing-theory/SKILL.md` — 排队论建模分析（13 个排队点 M/M/1 M/M/c M/G/1 端到端延迟估算）
- `code-review/SKILL.md` — GitHub PR 代码审查（多 agent 并行 + 置信度打分）
- `review/SKILL.md` — 本地代码审查（git diff 多维度分析）
- `clear-proposals/SKILL.md` — 清理历史方案（按 today/month/year）
- `plan-feature/SKILL.md` — 新功能设计方案（基础设施复用 + 知识库设计模式）
- `qa/SKILL.md` — Mooncake 快速问答
- `history/SKILL.md` — 优化会话记录说明

## 与新项目对接

当用户要求对非 Mooncake 项目进行优化分析时：
1. 告知用户当前支持 Mooncake
2. 询问目标项目的仓库地址和本地路径
3. 按 CLAUDE.md 中的「添加新目标项目的规范」创建项目目录和路由文件
4. 为新项目生成 repo-map.md（基于对目标仓库的初步探索）

## ⚠️ 后置操作：提交并推送变更（必须最后执行）

**在完成所有变更（新建方案、更新 KNOWLEDGE.md、追加优化记录等）之后，执行以下两步流程：**

### Step 1: 展示变更摘要并获取用户确认

在执行 commit + push 之前，**必须**先向用户展示变更摘要表格：

```markdown
| 文件 | 操作 | 说明 |
|------|------|------|
| proposals/optimize/xxx.md | 新建 | 优化方案: <标题> |
| mooncake/store/storage-backend/KNOWLEDGE.md | 更新 | 新增 N 组已验证的优化目标 |
| history/optimization-log.md | 更新 | 追加本次会话记录 |
```

然后使用 `AskUserQuestion` 询问用户是否同意提交：

```
question: "是否提交以上变更？"
options:
  - "提交并推送" (Recommended)
  - "仅提交不推送"
  - "放弃本次变更"
```

**仅在用户选择「提交并推送」或「仅提交不推送」后才执行 git 操作。**

### Step 2: 在用户同意后执行

```bash
cd <本 skill 所在目录> && git add -A && git diff --cached --stat && git commit -m "<变更摘要>" && git push
```

**规则**：
- `<变更摘要>` 格式：`"proposals(optimize): <方案标题>"` 或 `"proposals(feature): <方案标题>"` 或 `"knowledge: update <组件> optimization targets"` 或 `"skill: <变更描述>"`
- **仅在 `git diff --cached` 非空时执行**（有实际变更才 commit），无变更则跳过并告知用户
- push 失败时告知用户（网络问题等），但不重试、不阻塞
- commit 信息中**不要**加 `Co-Authored-By: Claude <noreply@anthropic.com>`
- 一整个调用中的所有变更聚合成 **1 次 commit**，不要多次提交
- 如果用户选择「仅提交不推送」，跳过 push 步骤

详细配置见 `config.md`。


## ⚠️ 维护规则

**当本 SKILL.md 有实质性更新（新增优化维度、修改分析流程、更新路由规则、新增领域知识映射等），必须同步检查并更新 `README.md` 中对应的项目结构、组件覆盖表、使用示例或模块说明。** 保持 README 始终反映最新的 skill 能力边界。
