# Rejected Proposals Whiteboard

被否决/放弃的方案记录（whiteboard）。**目的**：避免在后续 optimize / plan-feature 会话中重复提出已被否决的方案——记录已被排除的设计及其否决证据（借鉴 Jitskit(arXiv'26) 的 whiteboard 机制：accumulates designs the critic has already ruled out, along with the evidence, so the planner does not repropose approaches whose failure modes are already logged）。

## 记录格式

| 日期 | 子命令 | 组件 | 方案标题 | 否决阶段 | 否决理由 |
|------|--------|------|---------|---------|---------|
<!-- 追加新记录时，在表格末尾加一行。只追加，不修改历史。 -->

## 字段说明

| 字段 | 说明 |
|------|------|
| **否决阶段** | 预览确认（用户选「放弃本次方案」）/ 提交确认（用户选「放弃本次变更」）/ 事后清理（用户主动否决已生成方案） |
| **否决理由** | 用户给出的理由；用户未说明时记「用户未说明」+ 当时的方案预览要点，供后续判断 |

## 使用规则

1. **写入时机**：optimize / plan-feature 的 Phase 5 用户选「放弃本次方案」时，必须追加一行（含否决理由）
2. **查重时机**：Phase 4 查重时，除 `proposals/` 外必须同时查本文件——被否决过的方案**不得原样重提**
3. **重提条件**：仅当能针对否决理由给出新证据（新论文、代码演进、约束变化）时才可重提，且预览表格中必须明示「曾被否决：<理由>；本次不同点：<说明>」
