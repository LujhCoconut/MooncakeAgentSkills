# Mooncake Store — Client 已知优化目标

> **最后更新**: 2026-07-16
> **已分析次数**: 0

---

## 读写路径零拷贝验证与优化（初始分析）

### 当前状态
- `put_from()` / `get_into()` 声称零拷贝，使用预注册 buffer
- 需验证整条路径是否真的无 CPU 拷贝

### 潜在优化方向
- 端到端 trace 验证数据拷贝次数
- GPU buffer 的 pin/unpin 缓存策略
- 小对象的 metadata 查询开销优化（可能大于数据传输）
- Read pipelining（多个 get 请求复用 RDMA 连接）

### 相关领域知识
- `performance/system-tuning/KNOWLEDGE.md`
- `performance/gpu-ai-performance/KNOWLEDGE.md`

---

## 连接池与多路复用（初始分析）

### 当前状态
- 每个 client 独立 `setup()` 创建连接
- 需验证连接建立开销和数量上限

### 潜在优化方向
- Client-side connection pool（复用已有连接）
- 多路复用：单连接承载多个并发请求
- 连接预热（提前建立，减少首次请求延迟）
- 自适应连接数（根据负载动态调整）

### 相关领域知识
- `operations/cloud-infrastructure/KNOWLEDGE.md`
- `performance/concurrency/KNOWLEDGE.md`

---

## 批量操作调度优化（初始分析）

### 当前状态
- `batch_put_from()` / `batch_get_into()` 支持批量
- 需验证内部调度策略和并发度

### 潜在优化方向
- 自适应批量大小（根据对象大小和链路状态）
- 批量内部分排序（大对象优先？小对象优先？）
- 批量失败的部分成功语义明确化
- 跨 batch 的 pipeline（batch N 提交时 batch N-1 还在执行）

### 相关领域知识
- `performance/optimization-paradigms/KNOWLEDGE.md`
- `algorithms/resource-scheduling/KNOWLEDGE.md`

---

## P2P Store 优化（初始分析）

### 当前状态
- `p2p_store.cpp` 支持临时对象的 P2P 共享
- 需验证发现机制和传输路径

### 潜在优化方向
- Peer 发现的去中心化（DHT / gossip）
- P2P 传输的直连优化（绕过 Master）
- 热点对象的 P2P 预共享（proactive replication）
- P2P 的安全边界控制

### 相关领域知识
- `network/os-networking/KNOWLEDGE.md`
- `architecture/distributed-systems/KNOWLEDGE.md`
