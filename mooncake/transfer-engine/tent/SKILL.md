# Mooncake Transfer Engine — TENT 运行时优化

TENT (Transfer Engine NEXT) 是 Transfer Engine 的下一代运行时，引入动态传输选择、遥测驱动的切片调度、多路径并行传输等高级特性。

## 源代码地图

| 文件 | 说明 |
|------|------|
| `mooncake-transfer-engine/tent/runtime/slice_scheduler.cpp` | 多路径切片调度器 |
| `mooncake-transfer-engine/tent/runtime/transport_selector.cpp` | 动态传输协议选择 |
| `mooncake-transfer-engine/tent/runtime/telemetry_collector.cpp` | 遥测数据收集 |
| `mooncake-transfer-engine/tent/config/tent_config.yaml` | TENT 配置模板 |
| `mooncake-transfer-engine/tent/benchmark/tent_bench.cpp` | TENT 基准测试 |
| `mooncake-transfer-engine/include/tent/tent_runtime.h` | TENT 运行时 API |
| `mooncake-transfer-engine/include/tent/tent_config.h` | TENT 配置 API |

## 优化维度

### 1. 切片调度 (Slice Scheduling)
- **代码入口**: `slice_scheduler.cpp`
- **当前状态**: Phase 1 基础实现，Phase 2 规划声明式调度
- **关键问题**:
  - 切片大小如何决定？（固定？基于 MTU？基于 RTT？自适应？）
  - 多路径间的切片分配策略？（均分？加权？拥塞感知？）
  - 切片排序和重组开销？（乱序到达的 buffer 管理）
  - Head-of-line blocking？（大切片阻塞小切片？）
  - Phase 2 声明式调度：用户声明 QoS 目标，调度器自动决策
- **Domain Knowledge 映射**:
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 调度算法
  - `performance/optimization-paradigms/KNOWLEDGE.md` — 自适应优化
  - `network/os-networking/KNOWLEDGE.md` — 多路径传输

### 2. 动态传输选择
- **代码入口**: `transport_selector.cpp`
- **当前状态**: 多协议环境下的动态选择
- **关键问题**:
  - 选择决策的输入信号？（RTT、带宽、拥塞、错误率、队列深度？）
  - 决策频率？（每次传输？每个 epoch？）
  - 决策延迟的可接受上限？
  - 是否支持部分选择？（不同切片走不同协议）
- **Domain Knowledge 映射**:
  - `algorithms/resource-scheduling/KNOWLEDGE.md` — 在线决策
  - `network/os-networking/KNOWLEDGE.md` — 多路径调度

### 3. 遥测与自适应
- **代码入口**: `telemetry_collector.cpp`
- **当前状态**: 基础遥测收集
- **关键问题**:
  - 遥测数据的粒度和采样率？（逐包？逐 batch？抽样？）
  - 遥测的存储和查询？（内存循环 buffer？时序数据库？）
  - 遥测到调度决策的闭环延迟？
  - 是否支持异常检测和自动告警？
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md`
  - `performance/profiling-methodology/KNOWLEDGE.md`

### 4. 配置管理与调优
- **代码入口**: `tent_config.yaml`, `tent_config.h`
- **关键问题**:
  - 配置项的数量和复杂度？（操作者是否容易出错？）
  - 配置的验证和合法性检查？
  - 是否支持运行时配置热更新？
  - 自动调优（auto-tuning）的可能性？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md`

### 5. 链路自愈 (Phase 2)
- **规划中**: 链路层故障检测和自愈
- **关键问题**:
  - 链路故障的检测延迟？
  - 故障后的流量切换策略？（全量切换 vs. 部分降级？）
  - 链路恢复后的流量回切？
  - 自愈与上层应用的重试如何协调？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md`
  - `architecture/distributed-systems/KNOWLEDGE.md`

## 已知优化目标

参见 `KNOWLEDGE.md`。
