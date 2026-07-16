# Mooncake Transfer Engine — TENT 运行时优化

TENT (Transfer Engine NEXT) 是 Transfer Engine 的下一代运行时——引入动态传输选择、遥测驱动的切片调度、多路径并行等高级能力。从通信系统视角看，TENT 是「传输控制层」——类比 TCP 拥塞控制，但工作在应用层、面向异构多路径。

> **理论根基**: 参见 `../architecture.md` §3.4 (多路径调度问题)、§3.5 (通信系统优化维度)、§5 (三视角交叉点③)

## 通信系统视角

### TENT 在通信系统中的定位

```
传统 TCP 栈:                     TENT:
┌──────────┐                    ┌──────────┐
│   应用    │                    │   应用    │
├──────────┤                    ├──────────┤
│  Socket  │                    │  TENT    │ ← 切片调度、路径选择、遥测
├──────────┤                    ├──────────┤
│   TCP    │ ← 拥塞控制          │Transport │ ← RDMA/TCP/NVLink/...
├──────────┤                    ├──────────┤
│    IP    │                    │   NIC    │
├──────────┤                    └──────────┘
│   NIC    │
└──────────┘

区别: TCP 有一条拥塞控制逻辑，TENT 有 N 条路径的调度逻辑
     (N 条路径各有自己的拥塞状态，TENT 需要协调)
```

### 切片调度是一个在线优化问题

```
输入:
  - N 条路径, 每条路径 i 有: 带宽 B_i, 延迟 L_i, 拥塞窗口 C_i, 错误率 E_i
  - M 个待传输的数据块, 每个块有: 大小 S_j, deadline D_j (可选)

目标:
  minimize max(completion_time)  (最小化最大完成时间)
  s.t. path capacity constraints (不超过路径能力)

这是一个 NP-hard 的 scheduling 问题
→ 实际中需要使用贪心、启发式或在线学习
```

### 与经典问题的类比

| TENT 的问题 | 经典类比 | 相关算法 |
|------------|---------|---------|
| 切片大小选择 | TCP MSS / MTU discovery | Path MTU Discovery, BDP-based sizing |
| 多路径权重分配 | MPTCP 耦合拥塞控制 | LIA, OLIA, BALIA |
| 路径选择 | Multi-armed Bandit | Thompson Sampling, UCB |
| 自适应调度 | Adaptive Bitrate (ABR) | MPC, Pensieve (RL) |
| 故障检测 | Heartbeat / Lease | Φ-accrual failure detector |

## 源代码地图

| 文件 | 说明 | 子组件 |
|------|------|--------|
| `mooncake-transfer-engine/tent/runtime/slice_scheduler.cpp` | 多路径切片调度器 | tent |
| `mooncake-transfer-engine/tent/runtime/transport_selector.cpp` | 动态传输协议选择 | tent |
| `mooncake-transfer-engine/tent/runtime/telemetry_collector.cpp` | 遥测数据收集 | tent |
| `mooncake-transfer-engine/tent/config/tent_config.yaml` | TENT 配置模板 | tent |
| `mooncake-transfer-engine/tent/benchmark/tent_bench.cpp` | TENT 基准测试 | tent |
| `mooncake-transfer-engine/include/tent/tent_runtime.h` | TENT 运行时 API | tent |
| `mooncake-transfer-engine/include/tent/tent_config.h` | TENT 配置 API | tent |

## 优化维度

### 1. 切片调度 — 大小与分配策略

- **代码入口**: `slice_scheduler.cpp`

**切片大小的决定因素**：

```
optimal_slice_size = f(
    RTT,                # 往返时间 ↑ → 切片宜大（减少交互次数）
    path_bandwidth,     # 带宽 ↑ → 切片宜大（充分利用带宽）
    error_rate,         # 错误率 ↑ → 切片宜小（减少重传粒度）
    total_data_size,    # 总数据量 ↑ → 切片宜大（减少切片数量）
    concurrency         # 并发度 ↑ → 切片宜分散到多路径
)

经验公式: slice_size ≈ min(1MB, max(64KB, BDP / N_paths))
          其中 BDP = bandwidth × RTT (bandwidth-delay product)
```

**多路径分配策略**：

| 策略 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **均分 (Equal)** | 每条路径分配相等数量的切片 | 简单、公平 | 无视路径差异 |
| **加权 (Weighted)** | 按路径带宽比例分配 | 利用带宽差异 | 不感知拥塞 |
| **最短队列 (JSQ)** | 分配给完成队列最短的路径 | 自适应负载 | 需实时 CQ depth |
| **拥塞感知** | 基于 RTT/ECN 的权重动态调整 | 最优资源利用 | 需要准确的拥塞信号 |
| **预测性** | 基于历史完成时间的预测分配 | 学习优化 | 冷启动问题 |

**LLM 推理对切片调度的需求**：
- Prefill→Decode 的 KV cache 传输：大对象 (1-100MB)，完整性敏感（缺一块无法开始 Decode）
- → 大切片 (≥1MB) + 多路径并行 + 完成后统一通知
- Decode token 的 KV cache 传输：极小对象 (可能 <1KB)
- → 小切片或整消息 inline 传输

**关键问题**:
- 当前切片大小的决策：固定？自适应？基于什么？
- 多路径分配策略？
- 乱序到达的切片如何重组？（buffer 管理、内存开销）
- Head-of-line blocking 的处理？
- 不同 LLM 推理阶段（Prefill vs. Decode）是否使用不同的切片策略？

**Domain Knowledge 映射**:
- `algorithms/resource-scheduling/KNOWLEDGE.md` — 调度算法、online scheduling
- `network/os-networking/KNOWLEDGE.md` — 多路径传输、MPTCP
- `performance/optimization-paradigms/KNOWLEDGE.md` — 自适应优化

### 2. 动态传输选择 — 在线决策问题

- **代码入口**: `transport_selector.cpp`

**在线决策框架**：

```
┌────────────────────────────────────────────────┐
│           Transport Selector                    │
│                                                │
│  输入: 传输请求 (msg_size, src, dst, priority)    │
│        + 每个协议的状态 (RTT, bandwidth, errors)   │
│                                                │
│  决策: 选择哪个协议 (或哪几个协议混合)              │
│                                                │
│  反馈: 传输完成后的实际性能 (latency, throughput)   │
│        → 更新协议评分模型                         │
│                                                │
│  算法: Multi-armed Bandit / Contextual Bandit    │
│        Thompson Sampling / UCB / ε-greedy       │
└────────────────────────────────────────────────┘
```

**Contextual Bandit 建模**：

```
Context (特征向量):
  - 消息大小 (log scale)
  - 源-目标 NUMA 距离
  - 时间 (高峰期 vs. 低峰期)
  - 各协议的当前 queue depth

Action: 选择协议 p ∈ {RDMA, TCP, NVLink, ...}

Reward: -(传输延迟) 或 +(吞吐) 或二者的加权组合

目标: maximize cumulative reward
     = minimize average latency over time
```

**关键问题**:
- 选择决策的粒度？（每条消息独立决策 vs. 按 epoch batch 决策？）
- 探索 (exploration) vs. 利用 (exploitation) 的平衡？
- 决策延迟？（必须在多少 µs 内做出决定？）
- 冷启动时的默认选择？
- 上下文特征的工程实现？（实时获取 RTT、queue depth 的开销？）

**Domain Knowledge 映射**:
- `algorithms/resource-scheduling/KNOWLEDGE.md` — bandit algorithms, online learning
- `network/os-networking/KNOWLEDGE.md` — 协议选择

### 3. 遥测与闭环控制

- **代码入口**: `telemetry_collector.cpp`

**闭环控制系统**：

```
        ┌──────────────────┐
        │  Transport Layer │
        │  (RDMA/TCP/...)  │
        └────────┬─────────┘
                 │ 数据面
                 ▼
        ┌──────────────────┐
        │  Telemetry       │ ← 收集: latency, throughput, errors, QP depth...
        │  Collector       │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │  Policy Engine   │ ← 分析: 趋势检测, 异常检测, 预测
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │  Scheduler /     │ ← 决策: 调整权重, 切换路径, 改变切片大小
        │  Selector        │
        └────────┬─────────┘
                 │ 控制面
                 ▼
        ┌──────────────────┐
        │  Transport Layer │
        └──────────────────┘
```

**闭环的延迟要求**：
- 拥塞信号 (ECN/RTT): 需要 RTT 级别的反馈（微秒到毫秒）
- 路径状态 (错误率/带宽): 可以用秒级的统计
- 调度策略调整: 根据调整频率的 trade-off（太快 → 震荡，太慢 → 滞后）

**关键问题**:
- 遥测数据的采样粒度？（每消息？每 N 条消息？抽样？）
- 遥测数据的存储？循环内存 buffer？还是持久化到时序 DB？
- 异常检测的阈值和方法？
- 闭环延迟？从事件发生到调度策略调整的总时间

**Domain Knowledge 映射**:
- `operations/monitoring-observability/KNOWLEDGE.md` — 遥测、metrics
- `performance/profiling-methodology/KNOWLEDGE.md` — profiling 方法

### 4. 声明式 QoS（Phase 2 前瞻）

- **规划中**: Phase 2 声明式调度

**声明式 vs. 命令式调度**：

```
命令式 (当前):
  "使用 RDMA, 切片大小 1MB, 4 条路径均分"
  → 用户需要理解底层细节

声明式 (Phase 2):
  "这条传输的 p99 延迟 < 10ms, 最小带宽 > 10GB/s"
  → 调度器自动选择协议、切片大小、路径分配
```

**声明式调度的挑战**：
- 如何将 QoS 目标转化为可执行的调度策略？
- 如何验证 QoS 约束在当前条件下可达？
- 违反 QoS 时的行为？

**关键问题**:
- QoS 约束的表达能力？（延迟/带宽/可靠性/优先级？）
- 约束冲突时的优先级？（延迟优先还是带宽优先？）
- 约束不可满足时的降级策略？

**Domain Knowledge 映射**:
- `algorithms/resource-scheduling/KNOWLEDGE.md` — QoS 调度
- `architecture/cloud-native/KNOWLEDGE.md` — SLA 管理

### 5. 链路自愈（Phase 2 前瞻）

- **故障检测**：从检测到链路故障到切换路径的时间（目标 < RTT × 3）
- **自愈策略**：
  - 快速故障 (QP error): 立即切换，无需等待
  - 慢速故障 (gradual degradation): 渐进式切换，避免误判
  - 链路恢复: 先 probe，确认稳定后渐变回切

**关键问题**:
- 故障检测 = 主动 (heartbeat) + 被动 (error event) 结合？
- 故障后的 in-flight 请求处理？（重传 vs. 丢弃 vs. 切换到其他路径）
- 链路恢复后的流量回切策略？（立即全部回切 vs. 渐进）

**Domain Knowledge 映射**:
- `operations/cloud-infrastructure/KNOWLEDGE.md` — 故障检测、自愈
- `architecture/distributed-systems/KNOWLEDGE.md` — 容错

## 已知优化目标

参见 `KNOWLEDGE.md`。
