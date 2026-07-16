# Mooncake Transfer Engine — Memory 内存管理优化

内存管理负责 RDMA 内存注册、NVLink 内存分配、零拷贝路径和 GPU 内存生命周期管理。内存注册 (pin) 是 RDMA 传输的关键路径，直接影响传输延迟和吞吐。

## 源代码地图

| 文件 | 说明 |
|------|------|
| `mooncake-transfer-engine/src/memory_location.cpp` | 内存位置抽象 (DRAM/VRAM/NPU) |
| `mooncake-transfer-engine/src/transfer_engine.cpp` | registerLocalMemory(), allocate_managed_buffer() |
| `mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp` | NVLink 专用内存分配器 |
| `mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.h` | NVLink 分配器接口 |
| `mooncake-transfer-engine/include/cuda_alike.h` | 跨 GPU 厂商统一内存抽象 |

## 优化维度

### 1. 内存注册 (RDMA Pin) 优化
- **代码入口**: `transfer_engine.cpp:registerLocalMemory()`
- **当前状态**: 支持 RDMA pin + GPUDirect + Ascend Direct
- **关键问题**:
  - 注册 (pin) 的延迟有多大？是否在关键路径上？
  - 是否支持 pin 缓存？（已 pin 的 region 复用）
  - 大区域 vs. 小区域 pin 的策略？（大区域一次 pin，小区域动态 pin）
  - GPUDirect 的 BAR1 大小限制？
  - 多 GPU 下的内存可见性？
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — 内存管理、DMA
  - `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU 内存

### 2. NVLink 内存分配器
- **代码入口**: `nvlink_allocator.cpp`
- **当前状态**: NVLink 专用分配器
- **关键问题**:
  - 分配/释放的碎片化？
  - 与 CUDA allocator 的共存和竞争？
  - 池化预热？（避免首次分配延迟）
  - 多 GPU 间的对称分配？（allocator per GPU or shared？）
- **Domain Knowledge 映射**:
  - `algorithms/concurrent-data-structures/KNOWLEDGE.md` — 并发分配器
  - `architecture/accelerators/KNOWLEDGE.md` — GPU 架构

### 3. 零拷贝路径
- **代码入口**: `gdr_transport.cpp` (GPUDirect RDMA), memory registration 路径
- **关键问题**:
  - GPU → NIC 的零拷贝路径是否始终启用？
  - CPU bounce buffer 的使用场景？
  - 跨 NUMA 的 GPU-NIC 数据路径延迟？
  - 零拷贝 vs. 拷贝的切换阈值？（小消息拷贝更快？）
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md`
  - `performance/gpu-ai-performance/KNOWLEDGE.md`

### 4. 跨 NUMA 内存访问
- **代码入口**: `memory_location.cpp`
- **关键问题**:
  - 内存分配时是否感知 NUMA topology？
  - 远端 NUMA node 的内存访问延迟是 local 的多少倍？
  - 是否支持内存迁移（move_pages）？
  - NIC 和 GPU 的 NUMA 亲和性绑定？
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — NUMA 优化
  - `architecture/memory-storage-hierarchy/KNOWLEDGE.md`

### 5. Memory 可观测性
- **关键指标**:
  - 内存注册延迟和数量
  - NVLink allocator 的分配/释放速率、碎片率
  - GPU memory 使用率、pin 比例
  - 跨 NUMA 访问比例和延迟
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
