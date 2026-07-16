# Mooncake Operations & SRE Optimization

跨组件的运维、监控、性能分析和可靠性优化。Operations 不限于单一组件，聚焦于端到端的可观测性、故障恢复和运维效率。

## 优化维度

### 1. 监控与可观测性
- **当前状态**: Mooncake 提供 `transfer_engine_topology_dump` CLI、TENT telemetry
- **关键问题**:
  - 是否有结构化的 metrics 导出？（Prometheus / OpenTelemetry？）
  - Transfer Engine 的关键延迟指标（submit→complete 的 p50/p99）
  - Store 的读写延迟和命中率监控
  - Master 的健康检查和告警
- **Domain Knowledge 映射**:
  - `operations/monitoring-observability/KNOWLEDGE.md` — 可观测性最佳实践
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 云原生监控

### 2. 基准测试方法论
- **代码入口**: `transfer_engine_bench`, `tent_bench`, `store_bench`
- **当前状态**: 有基本的 benchmark 工具
- **关键问题**:
  - Benchmark 是否覆盖关键场景？（单节点、跨节点、多协议、多 GPU）
  - Benchmark 的可重复性？（环境变量隔离、预热、统计方法）
  - 是否有 CI 中的性能回归检测？
- **Domain Knowledge 映射**:
  - `performance/profiling-methodology/KNOWLEDGE.md` — 性能分析方法论
  - `operations/os-performance-tuning/KNOWLEDGE.md` — 性能测试

### 3. 错误诊断与恢复
- **当前状态**: 各组件的错误处理逻辑
- **关键问题**:
  - RDMA 链路断开的检测时间和恢复流程？
  - Master failover 的客户端透明性？
  - 错误日志的可操作性？（包含足够上下文？）
  - 是否有诊断工具/脚本？
- **Domain Knowledge 映射**:
  - `common/diagnosis-playbooks/` — 诊断手册
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 故障恢复

### 4. 性能剖析
- **当前状态**: 需在运行时通过外部工具（perf, nsys, etc.）分析
- **关键问题**:
  - 是否有内置的 profiling 支持？（如 trace points, span）
  - 关键路径上的系统调用开销？
  - 内存分配器的性能特征？
- **Domain Knowledge 映射**:
  - `operations/os-performance-tuning/KNOWLEDGE.md` — OS 层性能分析
  - `performance/profiling-methodology/KNOWLEDGE.md` — profiling 方法

### 5. 配置管理
- **代码入口**: `mooncake_config.py`, 各 CMake flags
- **当前状态**: 环境变量 + CMake flags 双层配置
- **关键问题**:
  - 配置项的数量和复杂度？（操作者是否容易出错？）
  - 是否有配置验证？（非法组合的检测）
  - 配置热更新的支持？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 配置管理

### 6. 安全与隔离
- **当前状态**: Mooncake 主要用于可信的内部集群
- **关键问题**:
  - Transfer Engine 的内存隔离？（是否可以读取其他进程的注册内存？）
  - Store 的访问控制？（谁可以 put/get？）
  - 多租户间的数据隔离？
- **Domain Knowledge 映射**:
  - `security/os-security/KNOWLEDGE.md` — 安全隔离
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 多租户

## 分析工作流

Operations 分析通常涉及多组件，步骤：
1. 明确运维场景（如 "如何监控端到端延迟"）
2. 识别涉及的组件
3. 搜索现有的 metrics/logging/tooling
4. 检索领域知识中的最佳实践
5. 生成运维优化方案

## 已知优化目标

参见 `KNOWLEDGE.md`。


## ⚠️ 维护规则

**当本 SKILL.md 有实质性更新（新增优化维度、修改分析流程、更新路由规则、新增领域知识映射等），必须同步检查并更新 `README.md` 中对应的项目结构、组件覆盖表、使用示例或模块说明。** 保持 README 始终反映最新的 skill 能力边界。
