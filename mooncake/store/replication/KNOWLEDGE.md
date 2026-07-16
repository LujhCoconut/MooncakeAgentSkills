# Mooncake Store — Replication & Placement 已知优化目标

> **最后更新**: 2026-07-16
> **已分析次数**: 0

---

## 拓扑感知的副本放置（初始分析）

### 当前状态
- `replica_manager.cpp` 管理副本策略
- 需验证是否考虑 NUMA、rack、GPU-NIC 拓扑

### 潜在优化方向
- **NUMA 亲和性**: 优先将副本放在与请求方同一 NUMA node 的节点上
- **Rack/DC 反亲和性**: 确保副本不在同一故障域
- **GPU-NIC 拓扑感知**: 副本优先放在 RDMA 可达（低延迟）的节点
- **异构感知**: 根据节点 DRAM 容量/SSD 速度加权选择
- **动态重平衡**: 节点加入/退出时的副本自动迁移

### 相关领域知识
- `architecture/cloud-native/KNOWLEDGE.md`
- `algorithms/resource-scheduling/KNOWLEDGE.md`
- `performance/system-tuning/KNOWLEDGE.md` — NUMA

---

## 复制协议优化（初始分析）

### 当前状态
- KV cache 对象 immutable after Put，简化复制语义
- 需验证写入是同步还是异步复制

### 潜在优化方向
- 利用 immutability：无需并发控制，无需 read-repair
- Chain replication 替代全量同步（降低 Master 带宽压力）
- Lazy replication（先写入单副本，后台异步复制）
- Quorum 配置的可调节性（写 1 读 1 vs. 写 all 读 1）

### 相关领域知识
- `algorithms/distributed-consensus/KNOWLEDGE.md`
- `architecture/cloud-native/KNOWLEDGE.md`

---

## P2P 共享的去中心化发现（初始分析）

### 当前状态
- P2P Store 用于临时对象（checkpoint 等）共享
- 需验证 peer 发现是否依赖 Master

### 潜在优化方向
- DHT-based peer discovery（O(log N) 查找）
- Gossip 协议的成员管理（弱一致但高扩展）
- 热点对象的 proactive push（主动推送到多个 peer）
- P2P 传输路径的延迟优化（选最近的 peer）

### 相关领域知识
- `network/os-networking/KNOWLEDGE.md`
- `architecture/distributed-systems/KNOWLEDGE.md`

---

## Placement 可观测性与约束验证（初始分析）

### 当前状态
- 需验证是否有放置决策的审计和违规检测

### 潜在优化方向
- 放置决策的可追溯性（为什么放在节点 X？）
- 亲和性/反亲和性违规的实时检测
- 副本分布的均衡性指标（熵值）
- 放置决策延迟的监控

### 相关领域知识
- `operations/monitoring-observability/KNOWLEDGE.md`
