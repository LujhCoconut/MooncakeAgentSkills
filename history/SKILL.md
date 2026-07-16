# Optimization Session History

优化会话记录目录。每次执行 `/advanced-optimize` 生成方案后，在此记录。

## 日志格式

`optimization-log.md` 使用 Markdown 表格，每行一次优化会话：

```markdown
| 日期 | 目标项目 | 组件 | 优化问题 | 引用的领域知识 | 方案路径 | 可应用性 | 备注 |
|------|---------|------|---------|---------------|---------|---------|------|
| YYYY-MM-DD | Mooncake | transfer-engine | 降低 RDMA 延迟 | network/os-networking (TMO OSDI'26) | proposals/... | 需适配 | |
```

## 字段说明

| 字段 | 说明 |
|------|------|
| **日期** | 优化分析的日期 (YYYY-MM-DD) |
| **目标项目** | Mooncake / vLLM / ... |
| **组件** | transfer-engine / store / conductor / integrations / build-deploy / operations |
| **优化问题** | 用户提出的优化问题简述 |
| **引用的领域知识** | 从中检索到可用洞察的 KNOWLEDGE.md 路径和论文 |
| **方案路径** | 生成的方案文件在 `proposals/` 中的路径 |
| **可应用性** | 可直接应用 / 需适配 / 仅参考 / 不适用 |
| **备注** | 其他说明（如"未找到适用论文"、"已存在类似方案"等） |

## 更新规则

1. 每次 `/advanced-optimize` 生成方案后，在日志末尾追加一行
2. 如果搜索领域知识后未找到可应用的洞察，仍然记录（可应用性 = "不适用"，备注说明原因）
3. 不要删除或修改历史记录（只追加）

## 统计用途

`optimization-log.md` 可用于统计：
- 最常被优化的组件
- 哪些领域知识被引用最频繁
- 哪些论文洞察落地率最高


## ⚠️ 维护规则

**当本 SKILL.md 有实质性更新（新增优化维度、修改分析流程、更新路由规则、新增领域知识映射等），必须同步检查并更新 `README.md` 中对应的项目结构、组件覆盖表、使用示例或模块说明。** 保持 README 始终反映最新的 skill 能力边界。
