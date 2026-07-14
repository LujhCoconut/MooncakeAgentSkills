# Mooncake Build & Deploy — 已知优化目标

本文档记录构建与部署组件中已识别的优化机会。

> **最后更新**: 2026-07-14
> **已分析次数**: 0 (初始种子文件)

---

## 构建并行化与增量编译（初始分析）

### 当前状态
- CMake 管理多子项目 (common, transfer-engine, store, integration)
- 多 GPU 厂商编译选项（CUDA/HIP/MUSA/MACA/MLU）

### 潜在优化方向
- 使用 CMake Unity Build 或 precompiled headers 加速全量编译
- ccache/sccache 集成到 CMake（CI 和本地开发）
- 按需编译 transport（默认只编译 TCP，RDMA/CUDA 可选）
- CUDA 代码的并行编译（NVCC parallel jobs）

### 相关领域知识
- `performance/system-tuning/KNOWLEDGE.md`
- `common/checklists/`

---

## CI/CD 流水线优化（初始分析）

### 当前状态
- GitHub Actions 流水线
- 需进一步分析具体的 workflow 配置

### 潜在优化方向
- 分层 cache (dependency → build → test)
- 矩阵构建的并行度调整
- 使用 self-hosted GPU runner 加速 GPU 相关测试
- PR 级别的增量测试（只跑变更影响的模块）

### 相关领域知识
- `operations/cloud-infrastructure/KNOWLEDGE.md`

---

## Wheel 打包优化（初始分析）

### 当前状态
- `scripts/build_wheel.sh` 生成 CUDA/非 CUDA 两种 wheel
- `mooncake-wheel/pyproject.toml` 定义包元数据

### 潜在优化方向
- 按 transport 拆分为 extras（`pip install mooncake[rdma]`）
- strip debug symbols 减小 wheel 体积
- 使用 manylinux 兼容的构建环境
- 发布到 PyPI 或私有 PyPI mirror

### 相关领域知识
- `operations/cloud-infrastructure/KNOWLEDGE.md`
