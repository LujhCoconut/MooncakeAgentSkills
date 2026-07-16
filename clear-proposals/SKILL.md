---
name: mooncake-agent-skills
description: 清理历史优化方案 — 按 today/month/year 时间范围删除 proposals，基于 git log 的最近 commit 时间判定
---

# clear-proposals — 清理历史方案

按时间范围删除 `proposals/` 目录下的优化方案文件，以 git log 中该文件的**最近一次 commit 时间**为依据。

> 入口：`/mooncake-agent-skills clear-proposals today|month|year`

## 时间范围定义

| 参数 | 范围 | git log 参数 |
|------|------|-------------|
| `today` | 今天 00:00:00 ～ 当前时刻 | `--since="$(date +%Y-%m-%d)T00:00:00"` |
| `month` | 本月 1 日 00:00:00 ～ 当前时刻 | `--since="$(date +%Y-%m)-01T00:00:00"` |
| `year` | 今年 1 月 1 日 00:00:00 ～ 当前时刻 | `--since="$(date +%Y)-01-01T00:00:00"` |

如果用户未指定参数或参数不在 `today/month/year` 中，提示正确的用法并退出。

## 执行流程

### Step 1: 查找候选文件

```bash
cd <本 skill 所在目录>

# 列出 proposals/ 下所有文件及其最近一次 commit 时间
git log --diff-filter=A --name-only --format="%H %ai" --since="<time_range>" -- proposals/
```

用 `git log --follow --format="%ai" -- <file>` 获取每个 proposal 文件的最近 commit 时间，筛选落在时间范围内的文件。

**简化命令**（推荐）：
```bash
# 获取所有 proposal 文件列表
ls proposals/*.md 2>/dev/null

# 对每个文件，检查其最近一次 commit 时间是否在范围内
for f in proposals/*.md; do
  last_commit=$(git log -1 --format="%ai" -- "$f" 2>/dev/null)
  if [ -n "$last_commit" ]; then
    echo "$last_commit $f"
  fi
done | sort -r
```

然后在 Python 或其他语言中解析日期，过滤出符合时间范围的文件。

### Step 2: 展示摘要并确认

**必须**先展示候选文件列表的摘要表格，然后获取用户确认：

```markdown
| 文件 | 最后 commit 时间 | 文件大小 |
|------|-----------------|---------|
| proposals/mooncake-store-bucket-ssd-offloading-optimization-2026-07-16.md | 2026-07-16 15:30:00 | 12.3 KB |
| proposals/mooncake-transfer-rdma-latency-2026-07-15.md | 2026-07-15 10:00:00 | 8.1 KB |

共 N 个文件，合计约 X KB。
```

使用 `AskUserQuestion` 确认：

```
question: "确认删除以上 N 个方案文件？（操作不可撤销）"
options:
  - "确认删除"
  - "取消"
```

### Step 3: 执行删除

仅在用户选择「确认删除」后执行：

```bash
cd <本 skill 所在目录>
git rm proposals/<file1>.md proposals/<file2>.md ...
git commit -m "housekeeping: clear proposals <today|month|year> (N files)" && git push
```

**规则**：
- 如果 `git diff --cached` 为空（无文件匹配） → 告知用户「该时间范围内无方案文件」，不 commit
- commit 信息格式：`"housekeeping: clear proposals <today|month|year> (N files)"`
- commit 信息中**不要**加 `Co-Authored-By: Claude <noreply@anthropic.com>`
- push 失败时告知用户，但不重试、不阻塞
- 如果用户选择「取消」→ 不做任何操作，告知用户已取消

## 边界情况

| 场景 | 行为 |
|------|------|
| `proposals/` 目录为空 | 提示「无方案文件可清理」 |
| 时间范围内无匹配 | 提示「未找到 <范围> 内的方案文件」 |
| 文件已通过其他方式删除（git 中已 staged 为 deleted） | 跳过，不重复删除 |
| `proposals/` 目录不存在 | 提示后退出 |

## 维护规则

当本 SKILL.md 有实质性更新，必须同步检查并更新根 `README.md` 中的能力表和项目结构。
