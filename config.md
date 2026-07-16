# MooncakeAgentSkills 配置

本文件定义优化 skill 的 Git 同步配置、目标仓库路径与领域知识库路径。

## Git 仓库

| 配置项 | 值 |
|--------|-----|
| **远程仓库** | `git@github.com:LujhCoconut/MooncakeAgentSkills.git` |
| **默认分支** | `main` |
| **本地路径** | `~/.claude/skills/mooncake-agent-skills/`（安装为 Claude Code skill 时） |
| **开发路径** | 当前仓库路径 |

## 目标仓库路径

| 项目 | 本地路径 | 远程仓库 |
|------|---------|---------|
| **Mooncake** | `~/src/Mooncake`（默认，可通过环境变量 `MOONCAKE_REPO` 覆盖） | `https://github.com/kvcache-ai/Mooncake` |
| **domain_knowledge_agent** | `~/.claude/skills/domain-knowledge/` | `git@github.com:LujhCoconut/domain_knowledge_agent.git` |

## 自动同步策略

### 前置操作（每次 skill 被调用时自动执行）

在读取任何子目录 SKILL.md 之前，先执行：

```bash
cd <MooncakeAgentSkills 本地路径> && git pull --rebase
```

- 目的：确保本地 skill 与远程仓库一致，避免编辑冲突
- 失败处理：如果 pull 失败（网络问题、冲突等），**继续执行**，不阻塞用户请求，但告知用户同步失败

### 后置操作（每次生成优化方案后自动执行）

在生成新的优化方案并写入 `proposals/` 或更新 `KNOWLEDGE.md` 后，执行：

```bash
cd <MooncakeAgentSkills 本地路径> && git add -A && git diff --cached --stat && git commit -m "<变更摘要>" && git push
```

- 提交信息格式：`"proposals: <方案标题>"` 或 `"knowledge: update <组件> optimization targets"`
- **仅在 `git diff --cached` 非空时执行**（有实际变更才 commit），无变更则跳过
- push 失败时告知用户（网络问题等），但不重试、不阻塞
- commit 信息中**不要**加 `Co-Authored-By: Claude <noreply@anthropic.com>`
- 一整个调用中的所有变更聚合成 **1 次 commit**

### 冲突处理

如果 `git pull --rebase` 产生冲突：
1. 报告用户有冲突发生
2. 不要自行解决冲突
3. 继续执行用户的原始请求

## 变更频率

- 每次 `optimize` 子命令产生新的优化方案后自动 commit + push
- 新增优化目标到 KNOWLEDGE.md 后也会触发提交

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `MOONCAKE_REPO` | `~/src/Mooncake` | Mooncake 源代码仓库路径 |
| `DOMAIN_KNOWLEDGE_PATH` | `~/.claude/skills/domain-knowledge/` | 领域知识库路径 |
