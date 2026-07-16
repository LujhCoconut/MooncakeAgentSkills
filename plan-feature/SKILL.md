---
name: mooncake-agent-skills
description: 新功能设计方案 — 分析现有基础设施可复用性 + 检索知识库设计模式 → 生成设计方案
---

# plan-feature — 新功能设计方案

针对 Mooncake 尚未实现的大功能，分析现有基础设施的可复用性，结合领域知识库的设计模式，生成结构化设计方案。

> 入口：`/mooncake-agent-skills plan-feature "<功能需求描述>"`
> 详细流程见根 `SKILL.md` § plan-feature

## 与 optimize 的区别

| 维度 | `optimize` | `plan-feature` |
|------|-----------|---------------|
| **关注点** | 现有代码的改进优化 | 未实现功能的设计方案 |
| **Phase 2** | 组件级代码分析（当前实现有哪些问题） | 基础设施复用分析（现有接口/模式可复用哪些） |
| **Phase 4** | 评估论文洞察的可应用性 | 设计实现方案（复用清单、新增代码、独立程度判断） |
| **方案输出** | `proposals/optimize/` | `proposals/feature/` |
| **共用的部分** | Phase 1/3/4.5/5/6 流程和模板完全一致 | |

## 基础设施复用分析指南

当分析新功能需要实现什么时，按以下优先级判断实现方式：

### 优先级 1: 可直接复用现有接口

如果现有接口/抽象类已经覆盖了新功能的核心语义，只需新增 adapter/plugin：

- 例：`FileSystemAdapter` 接口 → 新增 `CloudObjectStoreAdapter` 即可对接云存储
- 例：`StorageBackendInterface` → 新增 `CloudStorageBackend` 并注册到 `CreateStorageBackend()` 工厂

识别方法：
- 搜索 `include/` 下的抽象类和虚接口
- 查看 `CreateStorageBackend()` 等工厂方法的后端注册模式
- 查看 `MasterServiceConfigBuilder` 等 builder 模式的配置扩展点

### 优先级 2: 需扩展接口

如果现有接口接近但缺少关键方法，优先考虑最小化扩展：

- 在现有接口中新增虚函数（提供默认空实现以保持向后兼容）
- 在现有配置 struct 中新增可选字段

### 优先级 3: 需独立实现

如果功能与现有代码无交集，需要独立的文件：

- 独立的头文件 + 源文件
- 独立的 CMake feature flag
- 独立的配置和初始化路径

## 方案模板

与 `optimize` 使用相同的模板结构（见根 `SKILL.md` § 方案输出模板），但标题使用 `# 设计方案:` 而非 `# 优化方案:`。
