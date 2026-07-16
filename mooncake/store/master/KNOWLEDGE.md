# Mooncake Store — Master 已知优化目标

> **最后更新**: 2026-07-16
> **已分析次数**: 0

---

## Master 高可用与故障转移（初始分析）

### 当前状态
- Master 依赖 ETCD/Redis/HTTP 作为 metadata 后端
- Issue #1035 (Store V3) 提到 key-based routing 和 HA 增强

### 潜在优化方向
- Master 多副本 + Raft 选主（摆脱对 ETCD 的强依赖）
- 基于 ETCD lease 的 leader election（利用已有 ETCD 基础设施）
- 故障转移期间的 "读取不中断" 策略（segment 分配暂停，但 get 继续）
- Split-brain 防护（fencing token）
- 客户端透明的 Master 切换（retry + reconnection）

### 相关领域知识
- `algorithms/distributed-consensus/KNOWLEDGE.md`
- `architecture/cloud-native/KNOWLEDGE.md`
- `operations/cloud-infrastructure/KNOWLEDGE.md`

---

## 驱逐策略增强（初始分析）

### 当前状态
- 驱逐基于 TTL (hard 5s, soft-pin 30min) + 容量限制
- 对象 immutable after Put，不支持部分驱逐

### 潜在优化方向
- 引入 KV cache 价值模型：前缀层 > 中间层 > 尾部层（前缀被命中概率最高）
- 访问频率感知驱逐（LFU / ARC / LIRS）
- 基于 token 位置的加权 TTL（越靠前的 token TTL 越长）
- Layer-wise 驱逐（整层驱逐而非单对象，减少碎片）
- 驱逐的渐进性（提前标记、延迟删除，等 reader 完成）

### 相关领域知识
- `algorithms/resource-scheduling/KNOWLEDGE.md`
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md`

---

## Segment 分配优化（初始分析）

### 当前状态
- Master 集中分配 segment，1024 分片

### 潜在优化方向
- 分配请求的批量处理（减少 Master RPC 次数）
- 基于节点容量的加权分配（异构集群）
- 预分配池（减少分配延迟）
- 分配失败时的指数退避重试

### 相关领域知识
- `algorithms/resource-scheduling/KNOWLEDGE.md`
- `architecture/cloud-native/KNOWLEDGE.md`

---

## 租约管理优化（初始分析）

### 当前状态
- hard TTL 5s, soft-pin 30min
- client 崩溃后租约需等 TTL 过期

### 潜在优化方向
- Client 主动心跳续租（减少 "假死" 导致的租约过期）
- 基于 client 健康状态的动态 TTL（健康 client 给更长 TTL）
- 优雅关闭时主动释放租约（加速回收）
- 租约续期的批量处理（减少 Master 压力）

### 相关领域知识
- `architecture/distributed-systems/KNOWLEDGE.md`
- `algorithms/distributed-consensus/KNOWLEDGE.md`
