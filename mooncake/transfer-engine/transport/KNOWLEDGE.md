# Mooncake Transfer Engine — Transport 已知优化目标

> **最后更新**: 2026-07-16
> **已分析次数**: 0

---

## RDMA 多轨 (Multi-Rail) 与拥塞控制（初始分析）

### 当前状态
- RDMA 实现在 `rdma_transport.cpp`
- 需验证：单 QP vs. 多 QP？是否有 congestion control？

### 潜在优化方向
- 多 rail RDMA 的加权负载均衡（基于各 rail 的完成队列深度）
- DCQCN 参数自适应调优（ECN 标记阈值、速率降低因子）
- RDMA message size 自适应：< 256B 用 inline，> 用 RDMA write
- 多 QP 的 NUMA 感知绑定（QP 绑定到 local NIC）

### 相关领域知识
- `network/os-networking/KNOWLEDGE.md`
- `performance/network-performance/KNOWLEDGE.md`

---

## 协议降级与混合传输（初始分析）

### 当前状态
- `multi_transport.cpp` 支持多协议
- 需验证：是否有协议降级（RDMA→TCP fallback）？

### 潜在优化方向
- RDMA 链路故障的快速检测（< 100ms）和自动 TCP fallback
- fallback 期间的数据一致性保证
- 混合协议下的切片分配（延迟敏感走 RDMA，大块数据走 NVLink）
- 协议选择的历史学习（记住哪条路径对哪种大小的传输最快）

### 相关领域知识
- `network/os-networking/KNOWLEDGE.md`
- `algorithms/resource-scheduling/KNOWLEDGE.md`

---

## NVLink/CXL 带宽利用（初始分析）

### 当前状态
- NVLink transport 在 `nvlink_transport.cpp`
- CXL transport 在 `cxl_transport.cpp`

### 潜在优化方向
- NVLink 多链路的负载均衡（NVSwitch topology 感知）
- CXL 内存池化的延迟评估和优化
- NVLink-C2C 与 RDMA 的混合使用（GPU-GPU 用 NVLink，跨节点用 RDMA）

### 相关领域知识
- `architecture/accelerators/KNOWLEDGE.md`
- `architecture/memory-storage-hierarchy/KNOWLEDGE.md`
