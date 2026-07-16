---
name: advanced-optimize
description: 高级代码优化分析 skill。根据优化问题自动定位目标代码仓库的对应组件、检索领域知识库（domain_knowledge_agent）中的可复用论文洞察，评估可应用性并生成结构化优化方案。当前支持 Mooncake（KVCache 中心化 LLM 推理平台），覆盖传输引擎、分布式存储、缓存路由、框架集成、构建部署、运维监控等组件的优化分析。
---

# Advanced Optimize

基于领域知识的代码优化分析入口 skill。你的任务不是直接凭记忆给优化建议，而是先定位目标仓库的对应组件、阅读源代码、检索相关论文洞察，再评估可应用性并生成结构化方案。

## ⚠️ 前置操作：同步远程仓库（必须最先执行）

**在读取任何子目录文件或处理用户请求之前，必须先执行 git pull 同步：**

```bash
cd <本 skill 所在目录> && git pull --rebase
```

- 如果 pull 成功 → 继续处理用户请求
- 如果 pull 因网络问题失败 → 告知用户"⚠️ 无法同步远程仓库，继续使用本地版本"，不阻塞后续操作
- 如果 pull 产生冲突 → 告知用户"⚠️ git pull 产生冲突，请手动解决"，继续处理用户请求不阻塞

## 参数解析

用户输入格式：`/advanced-optimize <项目名> "<优化问题描述>"`

从用户输入中提取：
1. **目标项目**：第一个参数，如 `Mooncake`。如果只有一个参数（无引号包裹的问题描述），则将整个输入视为优化问题，目标项目默认为 Mooncake。
2. **优化问题**：描述想要优化的方向或具体问题。

示例：
- `/advanced-optimize Mooncake "降低 RDMA 传输延迟"` → 项目=Mooncake, 问题="降低 RDMA 传输延迟"
- `/advanced-optimize Mooncake "改进 KV cache 驱逐策略"` → 项目=Mooncake, 问题="改进 KV cache 驱逐策略"
- `/advanced-optimize "如何提升 Master 的 HA"` → 项目=Mooncake(默认), 问题="如何提升 Master 的 HA"

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

### Phase 4: 评估与生成

10. 对比论文洞察与当前代码实现
11. 评估每条洞察的可应用性：
    - **可直接应用**：当前实现存在明确 gap，论文方案可直接套用
    - **需适配**：核心思路可迁移，但需要针对 Mooncake 的架构做调整
    - **仅参考**：思路有价值但不完全匹配，提供设计参考
    - **不适用**：明确说明原因
12. 检查 `proposals/` 目录是否已有相关方案，避免重复
13. 读取 `<project>/<component>/KNOWLEDGE.md`，查看已记录的优化目标
14. 如果发现新的优化目标，更新 KNOWLEDGE.md

### Phase 5: 输出方案

15. 按方案模板生成优化方案到 `proposals/<project>-<component>-<slug>-<date>.md`
16. 更新 `history/optimization-log.md`
17. 向用户呈现方案摘要

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
- `history/SKILL.md` — 优化会话记录说明

## 与新项目对接

当用户要求对非 Mooncake 项目进行优化分析时：
1. 告知用户当前支持 Mooncake
2. 询问目标项目的仓库地址和本地路径
3. 按 CLAUDE.md 中的「添加新目标项目的规范」创建项目目录和路由文件
4. 为新项目生成 repo-map.md（基于对目标仓库的初步探索）

## ⚠️ 后置操作：提交并推送变更（必须最后执行）

**在完成所有变更（新建方案、更新 KNOWLEDGE.md、追加优化记录等）之后，必须 commit 并 push：**

```bash
cd <本 skill 所在目录> && git add -A && git diff --cached --stat && git commit -m "<变更摘要>" && git push
```

**规则**：
- `<变更摘要>` 格式：`"proposals: <方案标题>"` 或 `"knowledge: update <组件> optimization targets"` 或 `"skill: <变更描述>"`
- **仅在 `git diff --cached` 非空时执行**（有实际变更才 commit），无变更则跳过并告知用户
- push 失败时告知用户（网络问题等），但不重试、不阻塞
- commit 信息中**不要**加 `Co-Authored-By: Claude <noreply@anthropic.com>`
- 一整个调用中的所有变更聚合成 **1 次 commit**，不要多次提交

详细配置见 `config.md`。
