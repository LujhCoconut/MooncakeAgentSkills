# Mooncake Transfer Engine — Memory 已知优化目标

> **最后更新**: 2026-07-16
> **已分析次数**: 0

---

## RDMA 内存 Pin 缓存（初始分析）

### 当前状态
- `registerLocalMemory()` 注册内存用于 RDMA
- 需验证：是否有 pin 缓存？pin 延迟是否在关键路径？

### 潜在优化方向
- Pin cache：已 pin 的 memory region 缓存复用（LRU 淘汰）
- 预 pin：根据历史模式预测即将需要的内存区域
- Large page pin（减少 page table entry 数量，降低 pin 开销）
- On-demand pinning（延迟 pin，首次传输时再 pin）
- GPUDirect RDMA 的 BAR1 映射优化

### 相关领域知识
- `performance/system-tuning/KNOWLEDGE.md`
- `performance/gpu-ai-performance/KNOWLEDGE.md`

---

## NVLink 分配器碎片与并发（初始分析）

### 当前状态
- `nvlink_allocator.cpp` NVLink 专用分配器
- 需验证碎片率和并发性能

### 潜在优化方向
- Slab/Buddy allocator 减少碎片
- Per-thread cache 减少锁竞争
- 分配器的 NUMA 感知（local memory pool）
- 预热分配（避免首次传输的冷启动延迟）

### 相关领域知识
- `algorithms/concurrent-data-structures/KNOWLEDGE.md`
- `architecture/accelerators/KNOWLEDGE.md`

---

## 跨 NUMA 零拷贝延迟（初始分析）

### 当前状态
- `memory_location.cpp` 识别内存位置
- 需验证跨 NUMA 场景下的零拷贝路径

### 潜在优化方向
- GPU-NIC 的 NUMA 亲和性绑定（避免跨 NUMA 的 PCIe 访问）
- 跨 NUMA 的内存分配优先在 GPU 所在 NUMA node
- 量化跨 NUMA 的延迟惩罚表（供调度器决策用）

### 相关领域知识
- `performance/system-tuning/KNOWLEDGE.md`
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md`
