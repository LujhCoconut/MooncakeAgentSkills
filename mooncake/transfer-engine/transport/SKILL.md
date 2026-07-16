# Mooncake Transfer Engine — Transport 协议优化

传输协议层是 Transfer Engine 的核心——在异构高速网络上提供统一的数据传输接口。从通信系统视角看，这是 Mooncake 的「物理传输层 + 协议选择层」。

> **理论根基**: 参见 `../architecture.md` §3.1 (通信系统设计空间)、§3.2 (零拷贝路径)、§3.3 (RDMA 性能模型)

## 通信系统视角

### 传输协议的分层抽象

```
┌──────────────────────────────────────────────┐
│           Transfer Engine API                │ ← 统一接口
│   (submitTransfer, getTransferStatus, ...)   │
├──────────────────────────────────────────────┤
│             Transport Selector               │ ← 协议选择
│    (根据消息大小、QoS、拓扑选择最优协议)         │
├──────┬──────┬──────┬──────┬──────┬───────────┤
│ RDMA │ TCP  │NVLink│ EFA  │ CXL  │ NVMe-oF   │ ← 协议实现
├──────┴──────┴──────┴──────┴──────┴───────────┤
│         硬件层 (NIC, GPU, Switch, ...)         │
└──────────────────────────────────────────────┘
```

### 协议选择是一个多目标优化问题

```
选择协议 = argmin [w1×延迟 + w2×(1/带宽) + w3×CPU开销 + w4×pin开销]
           s.t. 硬件能力约束 (是否有 RDMA NIC? 是否同 NUMA? ...)
```

**不同消息大小的最优协议**：
- < 1KB: SHM (同节点) / RDMA inline (跨节点)
- 1KB - 64KB: RDMA send/recv (跨节点)
- 64KB - 1MB: RDMA write (单边，零拷贝)
- > 1MB: NVLink (GPU-GPU) / RDMA multi-rail (跨节点)

## 源代码地图

| 文件 | 说明 | 协议 |
|------|------|------|
| `mooncake-transfer-engine/include/transport/transport.h` | Transport 抽象基类 | 全部 |
| `mooncake-transfer-engine/src/transport/rdma_transport.cpp` | RDMA (IB/RoCE/eRDMA) | RDMA |
| `mooncake-transfer-engine/src/transport/tcp_transport.cpp` | TCP 传输 | TCP |
| `mooncake-transfer-engine/src/transport/nvlink_transport.cpp` | NVLink 传输 | NVLink |
| `mooncake-transfer-engine/src/transport/efa_transport.cpp` | AWS EFA 传输 | EFA |
| `mooncake-transfer-engine/src/transport/cxl_transport.cpp` | CXL 传输 | CXL |
| `mooncake-transfer-engine/src/transport/nvmeof_transport.cpp` | NVMe-oF 传输 | NVMe-oF |
| `mooncake-transfer-engine/src/transport/ascend_transport.cpp` | 华为 Ascend 直连 | Ascend |
| `mooncake-transfer-engine/src/transport/shm_transport.cpp` | 共享内存 (同节点) | SHM |
| `mooncake-transfer-engine/src/multi_transport.cpp` | 多协议混合传输 | 全部 |

## 优化维度

### 1. RDMA 传输深度优化

- **代码入口**: `rdma_transport.cpp`

**RDMA 协议栈分析**：

```
应用层 (submitTransfer)
    │
    ▼
Verbs API (ibv_post_send / ibv_poll_cq)
    │
    ▼
内核态 RDMA 驱动 (mlx5 / eRDMA)
    │
    ▼
NIC 硬件 (ConnectX-7 / 自定义)
```

**每层的优化杠杆**：

| 层 | 可优化项 | 预期收益 |
|----|---------|---------|
| **应用层** | WQE batching、doorbell batching | 10-30% 小消息延迟 |
| **Verbs API** | 减少 ibv_poll_cq 频率 (batch polling) | 5-15% CPU 开销 |
| **驱动** | 调整 QP 属性 (max_inline_data, max_send_wr) | 5-20% 延迟 |
| **硬件** | 多 QP、multi-rail、GDR | 线性带宽扩展 |

**消息大小自适应的 RDMA 操作选择**：

```
if msg_size <= inline_threshold:
    use IBV_WR_SEND_WITH_IMM  # 数据内联在 WQE 中，省一次 DMA read
elif msg_size <= 64KB:
    use IBV_WR_SEND           # 双边操作，简单
elif msg_size <= 1MB:
    use IBV_WR_RDMA_WRITE     # 单边操作，零拷贝远端
else:
    use multi-rail RDMA_WRITE # 多 QP 并行
```

**关键问题**:
- inline threshold 的当前值和调优依据？
- 多 QP 的负载均衡策略？（轮询？最短队列？加权？）
- GPUDirect RDMA (GDR) 的 DMA mapping 开销？
- RDMA memory pin 的缓存命中率？
- DCQCN 拥塞控制参数（ECN 标记阈值、速率降低因子）？

**Domain Knowledge 映射**:
- `network/os-networking/KNOWLEDGE.md` — RDMA 协议栈、用户态网络
- `performance/network-performance/KNOWLEDGE.md` — 网络调优、拥塞控制
- `performance/gpu-ai-performance/KNOWLEDGE.md` — GPUDirect RDMA

### 2. 多协议混合与动态选择

- **代码入口**: `multi_transport.cpp`, TENT `transport_selector.cpp`

**协议选择的决策框架**：

```
输入信号:
  - 消息大小 (message_size)
  - 源/目标节点 topology (NUMA, PCIe hierarchy)
  - 各协议的实时状态 (RTT, 拥塞窗口, QP depth, 错误率)
  - 用户 QoS (延迟优先 vs. 吞吐优先)

决策逻辑:
  if source == destination (同节点):
      → SHM
  elif GPU-GPU on same NVSwitch:
      → NVLink
  elif RDMA reachable AND message_size > RDMA_cutoff:
      → RDMA
  elif EFA available (AWS):
      → EFA
  else:
      → TCP (fallback)
```

**协议降级策略**：

```
优先级: NVLink > RDMA > TCP (性能从高到低)

降级触发:
  - RDMA QP 进入 error state → 立即降级到 TCP
  - RDMA 连续 N 次超时 → 降级
  - 检测到 RDMA 不可达 → 降级

升级触发:
  - 降级后定期探测 (probe) 原协议可用性
  - 探测成功后 → 渐进升级 (先少量流量, 确认稳定后全部切换)
```

**关键问题**:
- 协议选择的决策频率？（每次传输？每个 epoch？）
- 决策的输入信号是否充分？（是否需要更多 telemetry？）
- 协议降级的延迟？（从 RDMA 故障到 TCP fallback 的时间）
- 降级期间的 in-flight 请求如何处理？
- 是否支持混合模式（同一次传输的不同 slice 走不同协议）？

**Domain Knowledge 映射**:
- `network/os-networking/KNOWLEDGE.md` — 多路径传输
- `algorithms/resource-scheduling/KNOWLEDGE.md` — 在线决策、multi-armed bandit

### 3. TCP 传输优化

- **代码入口**: `tcp_transport.cpp`

**TCP 在 RDMA-centric 系统中的定位**：
- Fallback 路径（RDMA 不可用时）
- 小消息、管理消息（metadata 同步）
- 无需 GPU memory pin 的简单场景

**TCP 参数优化的上下文**：

| 参数 | 默认值 | 高性能场景建议 | 原因 |
|------|--------|--------------|------|
| **拥塞算法** | CUBIC | BBR | BBR 在高速网络下有更好的带宽利用率 |
| **Nagle** | ON | OFF | 小消息不需要等 ACK 攒包 |
| **TCP_NODELAY** | 0 | 1 | 禁用 Nagle |
| **buffer size** | ~128KB | BDP (bandwidth×RTT) | 充分利用带宽 |
| **SO_REUSEPORT** | OFF | ON | 多线程 accept 负载均衡 |

**io_uring 的适用性**：
- 当前 TCP transport 可能使用 epoll + read/write
- io_uring 可以将 read/write 系统调用批量提交 → 减少上下文切换
- 但 TCP fallback 的流量通常不大 → io_uring 的收益可能有限

**关键问题**:
- TCP 参数是否已经过调优？
- 是否使用 io_uring？
- TCP fallback 的切换延迟？
- TCP 连接池 vs. 短连接？

**Domain Knowledge 映射**:
- `performance/network-performance/KNOWLEDGE.md` — TCP 调优、BBR
- `operations/os-performance-tuning/KNOWLEDGE.md` — io_uring、epoll

### 4. NVLink / CXL / 新兴互连

- **代码入口**: `nvlink_transport.cpp`, `cxl_transport.cpp`

**互连技术对比**：

| 互连 | 带宽 (单链路) | 延迟 | 拓扑 | 使用场景 |
|------|-------------|------|------|---------|
| **NVLink 4.0** | 100 GB/s | ~1µs | NVSwitch all-to-all | GPU-GPU 通信 |
| **NVLink-C2C** | 450 GB/s | ~0.5µs | Point-to-point | Grace-Hopper |
| **CXL 3.0** | 64 GB/s | ~100ns | Switch fabric | 内存池化 |
| **PCIe 5.0** | 64 GB/s (x16) | ~500ns | Tree | CPU↔GPU↔NIC |
| **InfiniBand NDR** | 50 GB/s (x4) | ~1µs | Switch fabric | 跨节点 |

**Mooncake 中的互连使用策略**：
- GPU↔GPU (同节点): NVLink → 极致带宽
- GPU↔NIC (跨节点): PCIe → 出节点走网络
- GPU↔Remote GPU (跨节点): PCIe → NIC → RDMA Wire → NIC → PCIe
- 内存池 (未来): CXL → 跨节点内存共享

**关键问题**:
- NVLink 多链路的带宽聚合利用率？
- NVSwitch topology 下的最优 GPU pair 选择？
- CXL 内存池化对 Mooncake 的影响？（是否可以用远端 CXL 内存扩展 G2？）
- PCIe topology 是否与 NUMA 绑定保持一致？

**Domain Knowledge 映射**:
- `architecture/accelerators/KNOWLEDGE.md` — NVLink, CXL, GPU 互连
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md` — CXL 内存

### 5. Transport 可观测性

- **关键指标**:
  - 各协议的传输延迟 (p50/p99/p999)，按消息大小分组
  - 各协议的带宽利用率、in-flight 消息数
  - 协议选择的决策日志（为什么选 RDMA？为什么降级到 TCP？）
  - 协议切换次数、切换延迟
  - RDMA QP 状态转换次数（INIT→RTR→RTS→ERROR）
  - RDMA completion queue (CQ) depth、CQ overrun 计数
  - TCP retransmission rate、RTT 分布
  - 各协议的错误率（按错误类型）
  - 内存注册/注销延迟、注册 memory region 的总量和碎片情况
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。


## ⚠️ 维护规则

**当本 SKILL.md 有实质性更新（新增优化维度、修改分析流程、更新路由规则、新增领域知识映射等），必须同步检查并更新 `README.md` 中对应的项目结构、组件覆盖表、使用示例或模块说明。** 保持 README 始终反映最新的 skill 能力边界。
