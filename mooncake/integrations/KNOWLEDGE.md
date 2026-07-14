# Mooncake Framework Integrations — 已知优化目标

本文档记录框架集成组件中已识别的优化机会。

> **最后更新**: 2026-07-14
> **已分析次数**: 0 (初始种子文件)

---

## Python 绑定零拷贝路径（初始分析）

### 当前状态
- pybind11 绑定 TransferEngine 和 Store
- Python 侧有 `allocator.py` 管理内存
- 需验证：Tensor put/get 是否经过额外的 CPU 拷贝

### 潜在优化方向
- 使用 CUDA Array Interface (`__cuda_array_interface__`) 避免拷贝
- 支持 Python buffer protocol 实现真正的零拷贝
- pybind11 的 `return_value_policy::reference` 优化

### 相关领域知识
- `performance/system-tuning/KNOWLEDGE.md`
- `performance/gpu-ai-performance/KNOWLEDGE.md`

---

## vLLM Connector 内存优化（初始分析）

### 当前状态
- `MooncakeConnector` (KV Connector V1): Prefill↔Decode 分离的 KV cache 传输
- `MooncakeStoreConnector`: 跨实例 KV cache 共享

### 潜在优化方向
- 减少 Prefill→Decode 传输中的内存拷贝次数
- KV cache block 粒度的增量传输（只传变化的 blocks）
- GPU memory pin 缓存的复用策略
- 与 vLLM block manager 的预分配协作

### 相关领域知识
- `performance/gpu-ai-performance/KNOWLEDGE.md`
- `architecture/accelerators/KNOWLEDGE.md`

---

## SGLang HiCache 分层优化（初始分析）

### 当前状态
- Mooncake Store 作为 HiCache 的 L3 后端
- L1 (GPU HBM) / L2 (CPU DRAM) / L3 (Mooncake Store)

### 潜在优化方向
- L2→L3 迁移的批量预取（根据 attention pattern 预测需要的 KV cache）
- 前缀感知的 L3 placement（热门 prefix 保留在更近的存储节点）
- L3 读取延迟的隐藏（异步预取 + streaming）

### 相关领域知识
- `performance/gpu-ai-performance/KNOWLEDGE.md`
- `performance/storage-filesystem/KNOWLEDGE.md`
