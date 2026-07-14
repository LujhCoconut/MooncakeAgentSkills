# Mooncake Framework Integrations Optimization

Mooncake 通过 Python 绑定和框架连接器与主流 LLM 推理框架集成。包括 vLLM、SGLang、TensorRT-LLM、LMDeploy、LMCache 等。

## 源代码地图

参见 `mooncake/repo-map.md` § Framework Integrations。

### Mooncake 侧代码
- `mooncake-integration/engine.cpp` — TransferEngine 的 pybind11 绑定
- `mooncake-integration/store.cpp` — MooncakeDistributedStore 的 pybind11 绑定
- `mooncake-integration/async_store.py` — 异步 Store 辅助
- `mooncake-integration/allocator.py` — Python 内存分配器
- `mooncake-integration/mooncake_config.py` — 配置解析

### 框架侧代码（在各自仓库中）
- vLLM: `MooncakeConnector` (KV Connector V1), `MooncakeStoreConnector`
- SGLang: HiCache L3 Backend, EPD (Encode-Prefill-Decode)
- TensorRT-LLM: `mooncake_utils`
- LMDeploy: PD Backend Plugin
- LMCache: Remote Connector

### 集成文档
- `docs/source/getting_started/examples/sglang-integration-v1.md`
- `docs/source/getting_started/examples/lmcache-integration.md`
- `docs/source/getting_started/examples/vllm-integration/`

## 优化维度

### 1. Python 绑定开销
- **代码入口**: `engine.cpp`, `store.cpp` (pybind11)
- **当前状态**: pybind11 绑定，数据在 Python ↔ C++ 间传递
- **关键问题**:
  - Tensor 数据的序列化/拷贝开销（零拷贝路径是否通？）
  - pybind11 的函数调用开销？（关键路径上的调用频率）
  - GIL 的影响？（异步操作是否持有 GIL？）
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — 跨语言调用开销
  - `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU 数据搬运

### 2. vLLM Connector 内存效率
- **代码位置**: vLLM 仓库中的 `MooncakeConnector`
- **当前状态**: KV Connector V1（PD 分离），`MooncakeStoreConnector`（跨实例共享）
- **关键问题**:
  - Prefill→Decode 传输中的 KV cache 拷贝次数？
  - GPU memory 的 pin/unpin 开销？
  - 与 vLLM 的 block manager 的交互效率？
- **Domain Knowledge 映射**:
  - `performance/gpu-ai-performance/KNOWLEDGE.md` — vLLM 推理优化
  - `architecture/accelerators/KNOWLEDGE.md` — GPU 显存管理

### 3. SGLang HiCache 后端
- **代码位置**: SGLang 仓库中的 HiCache L3 Backend
- **当前状态**: Mooncake Store 作为 HiCache 的 L3 后端（分层 KV cache）
- **关键问题**:
  - L3 (Mooncake Store) 的读写延迟对端到端推理的影响？
  - HiCache 的分层策略（L1/L2/L3）是否最优？
  - EPD (Encode-Prefill-Decode) 的解耦效率？
- **Domain Knowledge 映射**:
  - `performance/gpu-ai-performance/KNOWLEDGE.md`
  - `performance/storage-filesystem/KNOWLEDGE.md`

### 4. 框架兼容性矩阵
- **当前状态**: 支持 5+ 框架，但集成方式各不相同
- **关键问题**:
  - 各框架对 Mooncake 功能的使用程度？（只用了 TE 还是也用了 Store？）
  - 版本兼容性管理？（框架升级是否 break Mooncake 集成？）
  - 是否有统一的集成测试？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 兼容性管理
  - `common/checklists/` — 集成测试检查清单

### 5. 异步接口优化
- **代码入口**: `async_store.py`
- **当前状态**: Python async wrapper
- **关键问题**:
  - asyncio event loop 是否成为瓶颈？
  - 异步操作的错误传播是否正确？
  - 是否支持取消/超时？
- **Domain Knowledge 映射**:
  - `performance/concurrency/KNOWLEDGE.md` — 异步编程模型

## 分析工作流

1. 确定分析的框架（vLLM / SGLang / ...）
2. 阅读对应的集成文档
3. 阅读 Mooncake 侧的 Python 绑定代码
4. 如有权限，阅读框架侧的连接器代码
5. 检索相关领域知识
6. 生成优化方案

> **注意**: 框架侧的代码不在 Mooncake 仓库中。分析时可能需要使用 WebFetch 获取框架连接器代码，或依赖文档和公开信息。

## 已知优化目标

参见 `KNOWLEDGE.md`。
