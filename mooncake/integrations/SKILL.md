# Mooncake Framework Integrations — LLM Serving 视角

框架集成层是 Mooncake 与 LLM 推理框架（vLLM、SGLang、TensorRT-LLM 等）的 **胶水层**。从 LLM Serving 视角看，这一层决定了 Mooncake 的能力能以多大开销融入到已有推理管线。

> **理论根基**: 参见 `../architecture.md` §4.3 (PD 分离)、§4.4 (端到端延迟模型)、§4.5 (优化杠杆)

## LLM Serving 视角

### 集成层在推理管线中的位置

```
推理框架 (vLLM / SGLang / ...)
    │
    ├── Scheduler (请求调度)
    ├── Model Runner (attention / FFN 计算)
    ├── Block Manager (KV cache block 管理)
    │       │
    │       ▼
    │   ┌──────────────────┐
    │   │ Mooncake Connector│  ← 集成层
    │   │  - TE connector   │     (框架侧的 glue code)
    │   │  - Store connector│
    │   └──────┬───────────┘
    │          │
    └──────────┼────────────────────
               │
    ┌──────────┴───────────┐
    │  Mooncake (C++ 侧)    │
    │  - Transfer Engine    │
    │  - Store              │
    └──────────────────────┘
```

**集成层的职责**：
1. 将框架的 KV cache block 映射为 Mooncake 的 put/get 操作
2. 管理 GPU memory 的生命周期（框架 vs. Mooncake 各自管理什么？）
3. 提供 Python API（pybind11 绑定）
4. 处理框架特定的约束（如 vLLM 的 block table）

### 跨语言边界的开销模型

```
框架 (Python) → pybind11 → C++ → Transfer Engine

每次跨语言调用的开销:
  pybind11 函数调用: ~100-500ns
  参数转换 (Tensor ↔ C++ pointer): ~50-200ns
  GIL 持有?: 如果持有 GIL，可能阻塞其他 Python 线程

如果每次 get/put 都走过这条路径:
  高频 decode (50 tokens/s × batch_size):
    50 × 16 requests × 2 calls (put + get) = 1600 calls/s
    → 每次调用 < 625µs 才不成为瓶颈
    → 当前开销是否在此范围内？
```

## 源代码地图

| 文件 | 说明 | 子组件 |
|------|------|--------|
| `mooncake-integration/engine.cpp` | TransferEngine 的 pybind11 绑定 → engine.so | integrations |
| `mooncake-integration/store.cpp` | MooncakeDistributedStore 的 pybind11 绑定 → store.so | integrations |
| `mooncake-integration/async_store.py` | 异步 Store 辅助类 | integrations |
| `mooncake-integration/allocator.py` | Python 侧内存分配器 | integrations |
| `mooncake-integration/mooncake_config.py` | 配置解析 (环境变量) | integrations |
| `mooncake-integration/allocator_ascend_npu.py` | 华为 Ascend NPU 分配器 | integrations |

### 框架侧连接器（在各自仓库中）
| 框架 | 连接器 | 关键文件 |
|------|--------|---------|
| vLLM | `MooncakeConnector` (KV Connector V1) | vLLM 仓库 |
| vLLM | `MooncakeStoreConnector` | vLLM 仓库 |
| SGLang | HiCache L3 Backend + EPD | SGLang 仓库 |
| TensorRT-LLM | `mooncake_utils` | TensorRT-LLM 仓库 |
| LMDeploy | PD Backend Plugin | LMDeploy 仓库 |

## 优化维度

### 1. Python↔C++ 边界的零拷贝路径

- **代码入口**: `engine.cpp`, `store.cpp`

**跨语言数据传输的四种路径**：

```
Path A (当前, 最可能的实现):
  GPU Tensor → pybind11::array_t → C++ pointer → Transfer Engine
  【有 pybind11 内部的 reference count 操作，但数据不拷贝】

Path B (如果使用了 .cpu()):
  GPU Tensor → CPU copy → pybind11 → C++
  【有显式的 GPU→CPU 拷贝】

Path C (理想路径):
  GPU Tensor → CUDA Array Interface (__cuda_array_interface__)
  → C++ CUDA pointer → Transfer Engine (GDR)
  【零拷贝，GPU 地址直接传递】

Path D (via DLPack):
  GPU Tensor → DLPack capsule → C++ → Transfer Engine
  【零拷贝，标准化格式】
```

**验证方法**：
- 在 pybind11 binding 中插入 trace 检查数据是否经过 CPU copy
- 使用 `nvidia-smi` 或 `nsys` 查看是否有 D2H (Device to Host) 传输

**关键问题**:
- 当前 Tensor put/get 使用什么路径？是否有隐式的 D2H 拷贝？
- pybind11 的 `return_value_policy` 是否正确？（避免不必要的拷贝）
- GIL 是否在传输期间释放？（不释放会阻塞 Python 推理线程）

**Domain Knowledge 映射**:
- `performance/system-tuning/KNOWLEDGE.md` — 跨语言调用、FFI 优化
- `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU data movement

### 2. vLLM Connector — KV Cache Block 管理

- **代码位置**: vLLM 仓库

**vLLM 的 block 模型**：
```
vLLM 将 KV cache 划分为固定大小的 blocks:
  Block size: 16 tokens (典型值)
  Block table: 每个 request 有 block_id → physical block 的映射

Mooncake 集成:
  当 block 从 GPU 驱逐 (vLLM side) → put(key, block_data) 到 Mooncake Store
  当 block 需要恢复 → get(key) 从 Mooncake Store
```

**与 vLLM Block Manager 的交互**：

| 操作 | vLLM 的动作 | Mooncake 的动作 | 优化机会 |
|------|-----------|----------------|---------|
| **Prefill 完成** | Blocks 写入 GPU | put() 到 Store | 可以异步 put（不等 Decode） |
| **Block eviction** | vLLM 驱逐 block | put() 到 Store (可能是 G2/G3) | 感知 block 的热度决定 G1/G2/G3 |
| **Block restore** | vLLM 请求 block | get() 从 Store | 预取 (prefetch adjacent blocks) |
| **Cross-instance** | 不同 vLLM 实例共享 | MooncakeStoreConnector | 跨实例的 block 一致性 |

**关键问题**:
- Block 粒度的 put/get 与 vLLM block manager 的交互是否高效？
- 是否支持增量 block 传输？（只传变化的 blocks）
- GPU memory 的 double-booking？（vLLM 和 Mooncake 都 pin 同一块 GPU memory？）

**Domain Knowledge 映射**:
- `performance/gpu-ai-performance/KNOWLEDGE.md` — vLLM, PagedAttention
- `architecture/cloud-native/KNOWLEDGE.md` — 跨实例共享

### 3. SGLang HiCache — 分层缓存集成

- **代码位置**: SGLang 仓库

**HiCache 的分层模型**：
```
L1: GPU HBM (vLLM-style blocks)    ← 最快, 最小
L2: CPU DRAM (SGLang managed)      ← 中
L3: Mooncake Store                  ← 最大, 跨节点共享
```

**Mooncake 作为 L3 后端的关键挑战**：
- L3 的延迟（RDMA + Store lookup）vs. L2 的延迟（local DRAM）→ 100×-1000× 差异
- 什么时候把数据推到 L3？什么时候从 L3 拉回来？
- L3 的命中对 Prefill 节省的收益 vs. L3 读取的延迟 → 需要成本-收益分析

**成本-收益模型**：
```
从 L3 获取 KV cache 的成本 = Store.get() 延迟 ≈ 50-500µs (RDMA)
从 L3 命中的收益 = 节省的 Prefill 计算时间

如果 Prefill 开销 > Store.get() 延迟:
  → 从 L3 获取是值得的
如果 Prefill 开销 < Store.get() 延迟:
  → 不如重新计算 Prefill

临界点: Prefill time(token N) ≈ Store.get() latency
  → 对于长 prompt (1000+ tokens), 从 L3 获取几乎总是值得的
  → 对于短 prompt (< 100 tokens), 重新计算可能更快
```

**关键问题**:
- HiCache 的 L2↔L3 迁移策略是否感知此成本-收益模型？
- L3 读取的预测（prefetch）是否可行？
- EPD (Encode-Prefill-Decode) 中不同阶段的 Mooncake 使用模式？

**Domain Knowledge 映射**:
- `performance/gpu-ai-performance/KNOWLEDGE.md` — HiCache, SGLang
- `performance/storage-filesystem/KNOWLEDGE.md` — 分层缓存

### 4. 异步接口优化

- **代码入口**: `async_store.py`

**异步的必要性**：
```
同步模式:
  put(key, data)  # 阻塞推理线程直到传输完成
  → 传输延迟直接增加到推理延迟上

异步模式:
  future = put_async(key, data)  # 立即返回
  → 推理可以继续
  → 传输在后台完成
  → future.wait() 在需要确认时同步

但从 LLM 推理的角度:
  Prefill 完成后, Decode 需要 KV cache
  → 如果 put 还没完成, Decode 就被阻塞了
  → 异步的价值在于: put 与 Prefill 后的其他工作 overlap
```

**异步接口的正确设计**：
- put 在 Prefill 期间就开始（streaming put？边生成边 put？）
- 或者利用 batch: 同一 batch 中多个 request 的 KV cache 可以流水线 put

**关键问题**:
- 当前的 async_store 是否真的实现了传输与推理的 overlap？
- 错误处理的正确性？（异步操作的异常传播）
- 是否支持取消/超时？

**Domain Knowledge 映射**:
- `performance/concurrency/KNOWLEDGE.md` — 异步编程模型
- `performance/optimization-paradigms/KNOWLEDGE.md` — 流水线

### 5. 跨框架的抽象层设计

- **代码位置**: 各框架的集成代码

**当前状态**: 每个框架有独立实现 → 维护成本高、优化不共享

**理想状态**: 统一的 Mooncake Serving Connector，框架只需要实现 thin adapter

```
┌────────────────────────────────────────────┐
│     Mooncake Serving Connector (统一层)      │
│  - KV cache block ↔ Mooncake key 映射        │
│  - GPU memory lifecycle 管理                 │
│  - 传输策略 (同步/异步/批量)                   │
│  - 健康检查和 fallback                        │
├────────────────────────────────────────────┤
│  Framework Adapters (thin):                  │
│  - vLLM adapter: block table ↔ Mooncake API │
│  - SGLang adapter: token pool ↔ API         │
│  - TRT-LLM adapter: ...                     │
└────────────────────────────────────────────┘
```

**关键问题**:
- 各框架连接器的代码重复程度？
- 是否可以提取共性到统一层？
- 统一层的性能开销？

**Domain Knowledge 映射**:
- `architecture/cloud-native/KNOWLEDGE.md` — 适配器模式
- `common/SKILL.md` — 集成模式

### 6. Integrations 可观测性

- **关键指标**:
  - Python↔C++ 调用频率、延迟分布
  - pybind11 的参数序列化/反序列化延迟
  - put/get 的端到端延迟（从框架调用到传输完成）
  - 异步操作的完成率、超时率
  - 各框架连接器的版本兼容性
  - GPU memory 的 ownership 冲突（框架 vs. Mooncake 同时使用）
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。


## ⚠️ 维护规则

**当本 SKILL.md 有实质性更新（新增优化维度、修改分析流程、更新路由规则、新增领域知识映射等），必须同步检查并更新 `README.md` 中对应的项目结构、组件覆盖表、使用示例或模块说明。** 保持 README 始终反映最新的 skill 能力边界。
