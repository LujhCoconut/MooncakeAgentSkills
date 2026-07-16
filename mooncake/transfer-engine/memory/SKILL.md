# Mooncake Transfer Engine — Memory 内存管理优化

内存管理负责 RDMA 内存注册、NVLink 分配器、零拷贝路径和 GPU 内存生命周期。从通信系统视角看，内存管理是传输路径的「预备阶段」——传输前的 memory pin 和分配开销决定了传输的启动延迟。

> **理论根基**: 参见 `../architecture.md` §3.2 (零拷贝路径)、§3.3 (RDMA 性能模型中的 memory pin 开销)

## 通信系统视角

### 内存管理在高速通信中的关键路径角色

```
一次 RDMA 传输的完整时间线:

│ Memory Pin │ DMA Mapping │ WQE Post │ DMA Read │ Wire Transfer │ CQ Poll │
│   ~1-5µs   │  ~0.5-1µs  │ ~0.2µs  │ ~0.5µs  │  ~1-10µs     │ ~0.3µs  │
│                                  │                                       │
│         ── 预备阶段 ──            │         ── 传输阶段 ──                 │
│     (可缓存/优化)                  │      (取决于硬件)                       │
└──────────────────────────────────┘
   预备阶段占小消息传输延迟的 30-70%
   对大消息, 传输阶段主导, 但预备阶段仍需优化 (影响吞吐)
```

**核心洞察**：Memory pin 是一次性开销，应该在注册后缓存复用，避免在每次传输的关键路径上重复。

### Memory Pin 的开销模型

```
pin_cost(n_pages) = c_fixed + n_pages × c_per_page

c_fixed: 固定开销 (syscall, 权限检查) ≈ 0.5µs
c_per_page: 每页开销 (page table entry 更新, TLB flush) ≈ 0.1µs

对比:
  4KB 对象 (1 page):    ~0.6µs
  1MB 对象 (256 pages):  ~26µs
  100MB 对象 (25600 pages): ~2560µs = 2.5ms ← 不可忽略!
```

**优化方向**：
- Huge pages (2MB/1GB): 减少 page table entry 数量 → pin 开销大幅降低
- Pin cache: 已 pin 的 region 复用 → 消除重复 pin 开销
- Pre-pin: 预测性地提前 pin → 隐藏 pin 延迟

## 源代码地图

| 文件 | 说明 | 子组件 |
|------|------|--------|
| `mooncake-transfer-engine/src/memory_location.cpp` | 内存位置抽象 (DRAM/VRAM/NPU) | memory |
| `mooncake-transfer-engine/src/transfer_engine.cpp` | registerLocalMemory(), allocate_managed_buffer() | memory |
| `mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp` | NVLink 专用内存分配器 | memory |
| `mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.h` | NVLink 分配器接口 | memory |
| `mooncake-transfer-engine/include/cuda_alike.h` | 跨 GPU 厂商统一内存抽象 | memory |

## 优化维度

### 1. 内存注册 (RDMA Pin) 优化 — 关键路径上的主要开销

- **代码入口**: `transfer_engine.cpp:registerLocalMemory()`

**Pin Cache 设计**：

```
Pin Cache:
  ┌───────────────────────────────────┐
  │ Key: (base_addr, length)          │
  │ Value: (mr_handle, pin_count, LRU timestamp) │
  │                                   │
  │ 操作:                              │
  │   lookup(addr, length) → mr or miss│
  │   insert(addr, length, mr)        │
  │   evict(LRU)                      │
  │                                   │
  │ 策略:                              │
  │   - 相同 (addr, length) → 直接复用  │
  │   - 子区域 (addr, length) 是已 pin   │
  │     父区域的一部分 → 拆分 MR?        │
  │     (RDMA 允许从 MR 的中间读写)       │
  └───────────────────────────────────┘
```

**大页优化**：
- 使用 hugetlbfs 或 THP (Transparent Huge Pages)
- 对 GPU memory 不适用 (GPU 使用自己的页表)
- 但对 CPU DRAM 中的 G2 存储有益

**GPUDirect RDMA 的特殊考虑**：
- GPU memory pin = 将 GPU VA 映射到 PCIe BAR
- BAR1 空间有限 (通常 256MB-16GB)，限制同时 pin 的 GPU memory 总量
- **BAR1 exhaustion 是一个可能被忽略的性能瓶颈**

**关键问题**:
- 当前是否有 pin cache？命中率？
- GPU memory pin 的 BAR1 限制是否曾被触发？
- Pin 的粒度？（每次传输 pin 新区域 vs. 大区域 pre-pin？）
- Huge pages 是否启用？
- 跨 NUMA 的 memory pin 是否有额外的延迟惩罚？

**Domain Knowledge 映射**:
- `performance/system-tuning/KNOWLEDGE.md` — huge pages, BAR1, DMA mapping
- `performance/gpu-ai-performance/KNOWLEDGE.md` — GPUDirect, GPU memory

### 2. NVLink 内存分配器 — GPU 侧的分配

- **代码入口**: `nvlink_allocator.cpp`

**GPU 内存分配器设计**：

```
GPU 内存分配面临的问题：
1. 碎片化: GPU 显存有限 (80GB)，频繁分配/释放 → 碎片
2. 并发: 多个 stream 可能同时分配
3. 延迟: cudaMalloc 有不可忽略的延迟 (10-100µs)
4. 与 CUDA 原生分配器的关系: cudaMalloc vs. 自己的 pool
```

**分配器设计对比**：

| 分配器 | 优点 | 缺点 |
|--------|------|------|
| **cudaMalloc** | 简单、CUDA 原生 | 延迟不可控 |
| **Slab Allocator** | 低碎片 (固定大小 slab) | 内部碎片 |
| **Buddy Allocator** | 灵活的大小 | 外部碎片 |
| **Pool Allocator** | 超低延迟 (O(1)) | 需要预热 |

**关键问题**:
- NVLink 分配器使用什么算法？碎片率？
- 分配/释放的并发安全性？(多 stream 竞争？)
- 是否支持池化预热？
- 与 CUDA 原生分配器的交互？

**Domain Knowledge 映射**:
- `algorithms/concurrent-data-structures/KNOWLEDGE.md` — 并发分配器
- `architecture/accelerators/KNOWLEDGE.md` — GPU 内存架构

### 3. 零拷贝路径完整性验证

- **代码入口**: `memory_location.cpp` → `gdr_transport.cpp`

**零拷贝路径的端到端验证方法**：

```
验证清单 (每次代码变更后检查):
  ☐ GPU VRAM → NIC 是否经过 CPU bounce buffer?
     检查: GPU memory 是否用 cudaMallocHost 分配的 pinned memory?
     或: GPUDirect RDMA 是否正确配置?

  ☐ 内存对齐是否满足 DMA 要求?
     DMA 通常要求 4KB 或 64B alignment

  ☐ BAR1 映射是否完成?
     检查 nvidia-smi 中的 BAR1 usage

  ☐ 跨 NUMA 时是否走了 QPI/UPI 而非 PCIe?
     → 可能需要 NUMA-aware 的内存分配

  ☐ 是否有隐式的 cudaMemcpy?
     Unified Memory (UM) 的 page fault 可能触发隐含拷贝
```

**零拷贝 vs. 拷贝的阈值决策**：
- 小消息 (< 4KB): CPU memcpy + TCP 可能比 RDMA pin + wire 更快
- 中消息 (4KB-64KB): RDMA inline (省 DMA read)
- 大消息 (> 64KB): RDMA write (真正零拷贝)

**关键问题**:
- 当前的 "零拷贝" 路径是否经过端到端验证？
- 跨 NUMA 场景下的零拷贝是否仍然零拷贝？（还是走了 QPI？）
- 不同 GPU vendor 的零拷贝能力差异？（NVIDIA/AMD/Ascend）

**Domain Knowledge 映射**:
- `performance/system-tuning/KNOWLEDGE.md` — DMA, 零拷贝
- `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU data movement

### 4. 跨 NUMA 内存访问 — 隐藏的延迟惩罚

- **代码入口**: `memory_location.cpp`

**NUMA 延迟矩阵 (典型值)**：

| 访问类型 | 延迟 | 带宽 | 相对惩罚 |
|---------|------|------|---------|
| Local DRAM | ~100ns | ~100 GB/s | 1× |
| Remote NUMA | ~150-200ns | ~60 GB/s | 1.5-2× |
| GPU VRAM (P2P) | ~500ns | ~50 GB/s (PCIe) | 5× |
| GPU VRAM (NVLink) | ~200ns | ~100 GB/s (NVLink 4.0) | 2× |

**NIC 和 GPU 的 NUMA 绑定**：
```
正确的绑定:
  NIC (mlx5_0) ↔ NUMA node 0 ↔ GPU 0-3 (在同一 PCIe domain)
  NIC (mlx5_1) ↔ NUMA node 1 ↔ GPU 4-7

错误的绑定:
  NIC (mlx5_0) ↔ NUMA node 0 ↔ 访问 GPU 4 (在 NUMA node 1)
  → 每次 DMA 都要跨 NUMA + 跨 PCIe root complex
  → 延迟增加 50-100%
```

**关键问题**:
- GPU 内存分配是否感知 NUMA？（local GPU 在哪个 NUMA node？）
- NIC 的中断亲和性是否绑定到正确的 NUMA node？
- CPU 端内存分配 (client allocator) 是否 NUMA-aware？
- 是否在日志/指标中暴露跨 NUMA 访问的比例？

**Domain Knowledge 映射**:
- `performance/system-tuning/KNOWLEDGE.md` — NUMA affinity, IRQ affinity
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 内存层次

### 5. Memory 可观测性

- **关键指标**:
  - 内存注册延迟 (p50/p99)、注册 region 数量、总 pin 内存
  - Pin cache 命中率、eviction 频率
  - BAR1 使用率（GPU side）
  - NVLink allocator: allocation rate, free rate, fragmentation ratio
  - 零拷贝路径的覆盖率（多少传输走了零拷贝？vs. bounce buffer？）
  - 跨 NUMA 内存访问的比例、延迟分布
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。


## ⚠️ 维护规则

**当本 SKILL.md 有实质性更新（新增优化维度、修改分析流程、更新路由规则、新增领域知识映射等），必须同步检查并更新 `README.md` 中对应的项目结构、组件覆盖表、使用示例或模块说明。** 保持 README 始终反映最新的 skill 能力边界。
