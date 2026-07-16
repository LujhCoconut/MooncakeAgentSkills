# Mooncake Transfer Engine — Transport 协议优化

传输协议层是 Transfer Engine 的核心，负责在多种底层协议（RDMA/TCP/NVLink/EFA/CXL 等）上提供统一的高性能数据传输接口。

## 源代码地图

| 文件 | 说明 |
|------|------|
| `mooncake-transfer-engine/include/transport/transport.h` | Transport 抽象基类 |
| `mooncake-transfer-engine/src/transport/rdma_transport.cpp` | RDMA (InfiniBand/RoCE/eRDMA) 实现 |
| `mooncake-transfer-engine/src/transport/tcp_transport.cpp` | TCP 传输实现 |
| `mooncake-transfer-engine/src/transport/nvlink_transport.cpp` | NVLink 传输实现 |
| `mooncake-transfer-engine/src/transport/efa_transport.cpp` | AWS EFA 传输实现 |
| `mooncake-transfer-engine/src/transport/cxl_transport.cpp` | CXL 传输实现 |
| `mooncake-transfer-engine/src/transport/nvmeof_transport.cpp` | NVMe-oF 传输实现 |
| `mooncake-transfer-engine/src/transport/ascend_transport.cpp` | 华为 Ascend 直连传输 |
| `mooncake-transfer-engine/src/transport/shm_transport.cpp` | 共享内存传输 |
| `mooncake-transfer-engine/src/multi_transport.cpp` | 多协议混合传输 |
| `mooncake-transfer-engine/include/transfer_engine.h` | TransferEngine 公共接口 (init, submitTransfer) |

## 优化维度

### 1. RDMA 传输优化
- **代码入口**: `rdma_transport.cpp`
- **关键问题**:
  - 单 QP vs. 多 QP 的选择？多 QP 的负载均衡策略？
  - RDMA 的 message size 自适应？（小消息用 inline，大消息用 RDMA write/read）
  - 拥塞控制？DCQCN 参数调优？
  - Multi-rail RDMA 的多路径调度？
  - GPUDirect RDMA (GDR) 的开启和优化？
  - RDMA 内存 pin 的缓存策略？（避免频繁 pin/unpin）
- **Domain Knowledge 映射**:
  - `network/os-networking/KNOWLEDGE.md` — RDMA、用户态网络
  - `performance/network-performance/KNOWLEDGE.md` — 网络调优

### 2. 多协议混合与选择
- **代码入口**: `multi_transport.cpp`, TENT `transport_selector.cpp`
- **关键问题**:
  - 协议选择的决策依据？（RTT？带宽？拥塞？硬件能力？）
  - 协议切换的代价和频率？
  - 是否支持协议降级（RDMA→TCP fallback）？
  - 混合协议下的数据分片策略？
- **Domain Knowledge 映射**:
  - `network/os-networking/KNOWLEDGE.md`
  - `algorithms/resource-scheduling/KNOWLEDGE.md`

### 3. TCP 传输优化
- **代码入口**: `tcp_transport.cpp`
- **关键问题**:
  - TCP 参数调优（拥塞算法 BBR vs. Cubic、buffer size、Nagle）
  - 是否使用 io_uring 或 epoll 减少系统调用？
  - 长连接 vs. 短连接的开销？
  - TCP 作为 RDMA fallback 的切换延迟？
- **Domain Knowledge 映射**:
  - `performance/network-performance/KNOWLEDGE.md`
  - `operations/os-performance-tuning/KNOWLEDGE.md`

### 4. NVLink / CXL / 新兴互连
- **代码入口**: `nvlink_transport.cpp`, `cxl_transport.cpp`
- **关键问题**:
  - NVLink 的带宽利用率？（理论 900 GB/s vs. 实际）
  - CXL 的延迟特征与 RDMA 的对比？
  - 多 GPU 下的 NVLink 路径选择（NVSwitch topology）
- **Domain Knowledge 映射**:
  - `architecture/accelerators/KNOWLEDGE.md`
  - `architecture/memory-storage-hierarchy/KNOWLEDGE.md`

### 5. Transport 可观测性
- **关键指标**:
  - 各协议的传输延迟 (p50/p99/p999)
  - 带宽利用率、重传率
  - RDMA QP 错误计数、completion queue 深度
  - 协议切换次数和延迟
  - 内存注册/注销延迟
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
