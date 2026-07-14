# Mooncake Transfer Engine / TENT Optimization

Transfer Engine 是 Mooncake 的数据平面，负责多协议、多 GPU 厂商的点对点数据传输。TENT (Transfer Engine NEXT) 是下一代运行时，引入动态传输选择、遥测驱动调度和多路径传输。

## 源代码地图

参见 `mooncake/repo-map.md` § Transfer Engine / TENT。

### 分析入口点

1. **协议选择与初始化**: `transfer_engine.cpp:init()` — 了解如何根据 metadata 协商传输协议
2. **内存注册**: `transfer_engine.cpp:registerLocalMemory()` — 了解 RDMA pin 和零拷贝路径
3. **批量提交**: `transfer_engine.cpp:submitTransfer()` — 了解批量和调度策略
4. **RDMA 实现**: `src/transport/rdma_transport.cpp` — 核心 RDMA 路径
5. **TENT 调度器**: `tent/runtime/slice_scheduler.cpp` — 多路径切片调度
6. **拓扑发现**: `topology.cpp` — GPU-NIC 亲和性检测

## 优化维度

### 1. 传输协议选择与多路径调度
- **代码入口**: `src/transport/`, `tent/runtime/transport_selector.cpp`
- **当前状态**: 支持 RDMA/TCP/NVLink/EFA/CXL 等多种协议，TENT 引入动态选择
- **关键问题**:
  - 协议选择决策是否考虑了实时拥塞信号？
  - 多路径调度是否感知链路质量差异？
  - 协议切换的开销和频率
- **Domain Knowledge 映射**:
  - `network/os-networking/KNOWLEDGE.md` — TCP/RPC 调度、RDMA 优化
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 多路径调度算法
  - `performance/network-performance/KNOWLEDGE.md` — 网络性能调优

### 2. 内存注册与零拷贝
- **代码入口**: `registerLocalMemory()`, `allocate_managed_buffer()`, `memory_location.cpp`
- **当前状态**: 支持 RDMA pin + GPUDirect + Ascend Direct
- **关键问题**:
  - 内存注册的延迟是否在关键路径上？
  - 是否有 pin/unpin 的缓存策略？
  - 跨 NUMA 节点的内存访问延迟
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — 内存管理、NUMA 优化
  - `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — 内存层次结构

### 3. 批量传输调度
- **代码入口**: `allocateBatchID()`, `submitTransfer()`, `getTransferStatus()`
- **当前状态**: 批量提交接口，需了解内部调度策略
- **关键问题**:
  - 批量大小如何选择？是否有自适应调整？
  - 批量内任务的调度顺序（FIFO? priority?）
  - 完成通知的开销
- **Domain Knowledge 映射**:
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 调度算法
  - `performance/optimization-paradigms/KNOWLEDGE.md` — 批处理优化
  - `performance/concurrency/KNOWLEDGE.md` — 并发模型

### 4. GPU-NIC 拓扑感知
- **代码入口**: `topology.cpp`, NVLink transport
- **当前状态**: 支持 GPU-NIC 拓扑发现
- **关键问题**:
  - 拓扑信息是否在调度决策中使用？
  - 是否支持跨 NUMA node 的 GPU-NIC 绑定？
  - 多 GPU 下的 NVLink 路径选择
- **Domain Knowledge 映射**:
  - `performance/gpu-ai-performance/KNOWLEDGE.md` — GPU 通信拓扑
  - `architecture/accelerators/KNOWLEDGE.md` — 加速器架构

### 5. NVLink 内存分配器
- **代码入口**: `nvlink-allocator/nvlink_allocator.cpp`
- **当前状态**: NVLink 专用内存分配器
- **关键问题**:
  - 分配/释放的碎片化问题？
  - 与 CUDA allocator 的竞争？
  - 是否可以池化预热？
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — 内存分配器优化
  - `algorithms/concurrent-data-structures/KNOWLEDGE.md` — 并发 allocator

### 6. TENT 动态切片调度
- **代码入口**: `tent/runtime/slice_scheduler.cpp`
- **当前状态**: 切片调度器（多路径并行传输）
- **关键问题**:
  - 切片大小如何决定？是否自适应？
  - 多路径间的负载分配策略？
  - 片间排序和重组的开销？
- **Domain Knowledge 映射**:
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 调度优化
  - `performance/optimization-paradigms/KNOWLEDGE.md` — 自适应优化

### 7. 错误恢复与故障转移
- **代码入口**: 各 transport 的错误处理路径
- **当前状态**: 各协议有独立的错误处理
- **关键问题**:
  - RDMA 链接断开的检测和恢复速度？
  - 传输失败的重试策略？（退避？立即？）
  - 是否支持透明的协议降级（RDMA→TCP）？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 故障恢复
  - `architecture/distributed-systems/KNOWLEDGE.md` — 分布式容错

## 分析工作流

对每个优化维度：
1. **Search** 源代码找到具体实现位置
2. **Read** 关键函数和数据结构
3. **Read** 对应的 KNOWLEDGE.md 文件
4. **对比** 论文洞察与当前实现
5. **评估** 按 `common/SKILL.md` 中的评估框架
6. **记录** 发现到 `KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
