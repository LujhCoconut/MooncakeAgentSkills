# Mooncake Quick Q&A — 知识库

> **最后更新**: 2026-07-16
> **覆盖范围**: 概念架构、环境配置、Transfer Engine、Store、框架集成、故障排查、性能调优

---

## 概念与架构

### Q: Mooncake 是什么？解决了什么问题？

Mooncake 是 Moonshot AI 开源的 LLM 推理服务平台，核心思路是 **KVCache 中心化的分离式架构**。解决的核心问题：

1. **Prefill 和 Decode 互相干扰**：Prefill 是 compute-bound，Decode 是 memory-bound，同节点部署会资源争抢 → PD 分离
2. **KV cache 跨请求无法复用**：每次新对话都要重新计算已有 system prompt 的 attention → 前缀缓存 + 分布式存储
3. **GPU 显存有限，DRAM/SSD 闲置**：GPU HBM 贵且小，但集群里大量 CPU DRAM 和 NVMe SSD 没用上 → 三级存储池化 (G1 GPU HBM / G2 CPU DRAM / G3 SSD)

FAST 2025 Best Paper，已加入 PyTorch 生态系统。

### Q: Mooncake 的 Prefill-Decode 分离 (PD Disaggregation) 是什么意思？

把 LLM 推理的两个阶段分开部署：

- **Prefill 节点**：处理输入 prompt，一次并行计算所有 token 的 attention → 产生 KV cache。GPU 密集型，需要高算力。
- **Decode 节点**：逐 token 生成输出，每次只算一个新 token，但要读全部历史 KV cache。内存/带宽密集型，需要大显存 + 高带宽。

Transfer Engine 负责把 Prefill 节点产生的 KV cache 传输到 Decode 节点。「XpYd」= X 个 Prefill 节点 + Y 个 Decode 节点。选择 X:Y 比例取决于 prompt 长度和生成长度的比值。

### Q: Mooncake 的 G1/G2/G3 三级存储是什么？

- **G1 (GPU HBM)**：GPU 显存，最快 (~µs)，最小 (几十 GB)，存最热的 KV cache
- **G2 (CPU DRAM)**：主机内存，较快 (~100ns local, ~1µs cross-NUMA)，中等 (几百 GB)，存温数据
- **G3 (NVMe SSD)**：本地 SSD，较慢 (~10-100µs)，最大 (几 TB)，存冷数据

数据在三级间自动迁移。Store 的 Master 负责管理哪些数据在哪个 tier。

### Q: Mooncake 和 vLLM/SGLang 的关系是什么？

Mooncake **不是**推理框架。它是推理框架的 **数据平面基础设施**：

- vLLM/SGLang 负责：模型加载、attention 计算、token 采样（推理逻辑）
- Mooncake 负责：KV cache 的跨节点传输和分布式存储（数据基础设施）

通过连接器（Connector）集成：
- vLLM: `MooncakeConnector` (KV V1), `MooncakeStoreConnector`
- SGLang: HiCache L3 Backend, EPD (Encode-Prefill-Decode)
- TensorRT-LLM: `mooncake_utils`

### Q: TENT 和 Transfer Engine 有什么区别？

- **Transfer Engine (TE)**：经典版本，提供多协议的基础传输能力。配置驱动，手动选择协议和参数。
- **TENT (Transfer Engine NEXT)**：下一代运行时。新增动态传输协议选择（根据实时遥测自动选协议）、多路径切片调度（大数据切成多个 slice 并行传输）、遥测驱动的闭环控制。通过 `MC_USE_TENT=1` 启用。

类比：TE = 手动挡（高性能但需要懂参数），TENT = 自动挡（自适应 + 智能调度）。

---

## 环境配置与安装

### Q: 如何安装 Mooncake？

```bash
# 1. 安装依赖
sudo bash dependencies.sh -y

# 2. 编译
mkdir build && cd build
cmake .. -DUSE_CUDA=ON -DWITH_STORE=ON -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install

# 3. Python wheel (可选)
./scripts/build_wheel.sh
pip install dist/mooncake_transfer_engine*.whl
```

前置：CMake 3.18+、GCC 10+ 或 Clang 14+、CUDA 12.8.1 (如果用 GPU)、RDMA 库 (libibverbs, librdmacm)。

### Q: Mooncake 支持哪些 GPU 厂商？

通过 `cuda_alike.h` 统一抽象，支持：
- **NVIDIA** (CUDA, `USE_CUDA=ON`)
- **AMD** (ROCm/HIP, `USE_HIP=ON`)
- **华为 Ascend** (`USE_ASCEND=ON`)
- **Moore Threads MUSA** (`USE_MUSA=ON`)
- **Cambricon MLU** (`USE_MLU=ON`)
- **MetaX MACA** (`USE_MACA=ON`)

每个厂商有对应的 CMake flag。默认只编译 TCP transport。

### Q: Mooncake 需要什么硬件？

**最低**：有网卡的 x86 服务器，TCP 模式即可运行（功能验证）

**推荐（生产环境）**：
- NVIDIA GPU (A100/H100/B200) + InfiniBand/RoCE 网卡
- GPU 和 NIC 在同一 PCIe root complex (GPUDirect RDMA 的前提)
- 如果有 NVSwitch → NVLink 传输可用
- 如果有 CXL 设备 → 可选 CXL 传输

### Q: 构建时常见 CMake 错误怎么解决？

| 错误 | 原因 | 解决 |
|------|------|------|
| `Could not find CUDA` | CUDA 未安装或 PATH 不对 | `export PATH=/usr/local/cuda/bin:$PATH` |
| `Could NOT find RDMA` | RDMA 库缺失 | `apt install libibverbs-dev librdmacm-dev` |
| `yalantinglibs not found` | submodule 未初始化 | `git submodule update --init --recursive` |
| `C++20 coroutines not supported` | 编译器太旧 | 升级到 GCC 10+ 或 Clang 14+ |
| `pybind11 not found` | submodule 未初始化 | `git submodule update --init extern/pybind11` |
| `WITH_STORE requires HTTP/ETCD/Redis` | Store 需要 metadata backend | 至少 `-DUSE_HTTP=ON` |

### Q: 如何选择 CMake 编译选项？

| 使用场景 | 推荐 CMake flags |
|---------|-----------------|
| 仅 TCP，快速验证 | `-DUSE_CUDA=OFF -DWITH_STORE=OFF` |
| GPU + RDMA，生产环境 | `-DUSE_CUDA=ON -DWITH_STORE=ON -DUSE_HTTP=ON` |
| 启用 TENT 实验 | `-DUSE_CUDA=ON -DUSE_TENT=ON` |
| AMD GPU | `-DUSE_HIP=ON` (替代 USE_CUDA) |
| 用 ETCD 做 metadata | `-DUSE_ETCD=ON` |
| 完整功能 | `-DUSE_CUDA=ON -DWITH_STORE=ON -DUSE_HTTP=ON -DUSE_REDIS=ON -DUSE_ETCD=ON -DUSE_TENT=ON` |

---

## Transfer Engine / RDMA

### Q: Transfer Engine 支持哪些传输协议？如何选择？

| 协议 | 适用场景 | 延迟 | 带宽 | 前提条件 |
|------|---------|------|------|---------|
| **RDMA** | 跨节点，低延迟，大数据 | ~1-5µs | 100-400 Gbps | InfiniBand/RoCE 网卡 |
| **TCP** | 通用，fallback | ~10-50µs | 受内核栈限制 | 无 |
| **NVLink** | 同节点 GPU↔GPU | ~0.5-1µs | 100-900 GB/s | NVSwitch/NVLink bridge |
| **EFA** | AWS 环境 | ~5-10µs | 100-400 Gbps | AWS EFA adapter |
| **CXL** | 内存池共享 | ~100ns-1µs | 32-64 GB/s | CXL 设备 |
| **SHM** | 同节点进程间 | ~0.1-0.5µs | 内存带宽 | 同一节点 |
| **NVMe-oF** | 存储分离 | ~10-100µs | 受 NVMe 限制 | NVMe-oF target |

选择原则：同节点用 SHM/NVLink；跨节点优先 RDMA；TCP 作为最后 fallback。TENT 可以自动选择。

### Q: RDMA 传输的延迟大概是多少？怎么优化？

RDMA 传输延迟 ≈ **1-5µs 固定开销** + **0.5-1ns/byte 传输时间**（取决于链路速度）。

优化方向：
1. **小消息用 inline**：消息 < `max_inline_data`（通常 256-4096 bytes）时数据嵌入 WQE 中，省一次 DMA read → 延迟降低 20-40%
2. **大消息用 RDMA write**：> 64KB 时用单边 RDMA write，绕过远端 CPU
3. **多 QP**：多个 Queue Pair 并行传输，带宽线性扩展
4. **GPUDirect RDMA**：GPU VRAM → NIC 直传，跳过 CPU → 延迟降低 50%+
5. **Memory pin 缓存**：已注册的内存区域复用，避免重复 pin 开销

具体调优参考：`mooncake/transfer-engine/transport/SKILL.md`

### Q: GPUDirect RDMA 怎么确认是否生效？

```bash
# 1. 检查 GPU-NIC 拓扑
transfer_engine_topology_dump

# 2. 确认 GPU 和 NIC 在同一 PCIe root complex
nvidia-smi topo -m | grep mlx5

# 3. 确认 BAR1 大小足够
nvidia-smi -q | grep -A2 "BAR1"

# 4. 运行时验证
# 使用 nsys/ncu 检查 GPU→NIC 是否有 D2H (Device to Host) 传输
# 如果有 D2H copy → GDR 未生效
```

**必要条件**：GPU 和 NIC 在同一 PCIe root complex 下，且支持 ACS redirect（或 ACS 已禁用）。跨 NUMA 的 GPU-NIC 通常不支持 GDR。

### Q: RDMA 连接报错 "cannot create QP" 或 "device not found"

常见排查步骤：

```bash
# 1. 确认 RDMA 设备存在
ibv_devinfo

# 2. 确认设备状态正常
ibstat

# 3. 确认权限 (非 root 可能需要设置 ulimit)
ulimit -l unlimited

# 4. 检查是否有足够的 QP 资源
cat /sys/class/infiniband/mlx5_0/device/hca_type

# 5. RoCE 模式下确认 GID 表
show_gids
```

如果 `ibv_devinfo` 无输出 → RDMA 驱动未安装或硬件不支持。

---

## Mooncake Store

### Q: Mooncake Store 怎么启动？

```bash
# 1. 启动 HTTP metadata server (最简单)
mooncake_http_metadata_server &

# 2. 启动 Master
mooncake_master \
  --metadata_server="http://localhost:8080" \
  --local_addr="192.168.1.100:50051"
```

然后客户端（推理框架侧）连接即可：
```python
store = MooncakeDistributedStore()
store.setup(
    local_addr="192.168.1.101:50052",
    metadata_url="http://192.168.1.100:8080",
    segment_size=256*1024*1024,  # 256MB per segment
)
```

### Q: Store 的 put/get 操作是否阻塞？性能如何？

- `put()` / `get()` 是 **同步**的（阻塞直到传输完成）
- `put_from()` / `get_into()` 支持 **零拷贝**（预注册 buffer，直接用 RDMA 传输）
- `batch_put_from()` / `batch_get_into()` 支持 **批量** 操作
- `async_store.py` 提供 Python 异步包装

典型性能（RDMA 环境）：
- 小对象 (<1MB) put/get: 50-200µs
- 大对象 (10-100MB) put/get: 1-10ms
- 零拷贝 get_into: 延迟约为 RDMA read + metadata lookup

### Q: Store 的 TTL 和驱逐 (Eviction) 怎么配置？

- **Hard TTL (默认 5s)**：client 必须在此时间内续期 (renew)，否则数据可能被驱逐。确保 decode 过程中定期续期。
- **Soft TTL (默认 30min)**：Hard TTL 过期后，数据进入宽限期。期间数据可读但可被容量压力驱逐。
- **容量驱逐**：当 segment 空间不足时，优先驱逐 soft TTL 已过期的对象。

调优建议：
- 如果 decode 经常超过 5s → 增加 hard TTL
- 如果 KV cache 需要在多轮对话间共享 → 增加 soft TTL
- 如果 DRAM 紧张 → 降低 soft TTL，让数据更快降级到 G3 (SSD)

### Q: Master 挂了对正在运行的推理有影响吗？

- **已有 KV cache 的读取 (get)**：**不受影响**。数据路径是 Client 直接 RDMA 读写存储节点，Master 不参与。
- **新 KV cache 的写入 (put)**：**受影响**。put 需要 Master 分配 segment，Master 挂 = 新 put 失败。
- **租约续期**：**受影响**。Master 挂 = 无法续期，已注册的 KV cache 在 hard TTL (5s) 后过期。

建议：部署 ETCD 或 Redis 作为 metadata backend，实现 Master 的 HA。

### Q: Store 的数据存在哪里？磁盘路径怎么看？

数据分布在集群各节点的 G2 (DRAM) 或 G3 (SSD) 上。Master 管理 segment 的分配，Client 侧不感知具体磁盘路径。

如需查看本节点被分配了哪些 segment：
- 通过 Master 的 API 查询 (如 HTTP metadata server)
- 或查看 Store 的 metrics 输出

---

## 框架集成

### Q: Mooncake 如何集成到 vLLM？

两种集成模式：

1. **KV Connector V1 (PD 分离)**：`MooncakeConnector` 负责 Prefill→Decode 的 KV cache 传输。配置 vLLM 的 disagg prefill 模式时启用。
2. **KV Cache 存储 (跨实例共享)**：`MooncakeStoreConnector` 支持将 KV cache 写入 Mooncake Store，不同 vLLM 实例间共享。

配置示例（详见 `docs/source/getting_started/examples/vllm-integration/`）。

### Q: SGLang 和 Mooncake 的 HiCache L3 后端是怎么工作的？

SGLang HiCache 的分层缓存：
- L1: GPU HBM (SGLang 管理)
- L2: CPU DRAM (SGLang 管理)
- L3: Mooncake Store (Mooncake 管理，跨节点共享)

当 SGLang 需要加速 Prefill 时：
1. 先查 L1 → 命中直接读
2. L1 miss → 查 L2 → 命中读到 L1
3. L2 miss → 通过 Mooncake Store 查 L3 → 命中读到 L2+L1
4. L3 miss → 完整 Prefill 计算

**配置关键参数**：L3 的延迟 (RDMA + Store lookup) vs. Prefill 重算的延迟。长 prompt (>1000 tokens) 走 L3 几乎总是值得的；短 prompt 重算可能更快。

### Q: Mooncake 可以不用 GPU，只用 CPU 跑吗？

可以。Transfer Engine 的 TCP transport 不需要 GPU。Store 的 client 也不需要 GPU。

但 LLM 推理的部分（Prefill / Decode 计算）仍然需要 GPU（由推理框架管理）。CPU-only 节点可以作为 DummyClient 访问 Mooncake Store 中的数据。

---

## 故障排查

### Q: Transfer Engine 报 "Cannot connect to metadata server"

```bash
# 1. 确认 metadata server 在运行
curl http://<metadata_server>:<port>/health

# 2. 确认网络可达
ping <metadata_server>

# 3. 检查地址格式
# metadata_server 格式: "http://host:port" 或 "etcd://host:port" 或 "redis://host:port"

# 4. 检查防火墙
iptables -L | grep <port>
```

### Q: RDMA 传输失败，有什么降级方案？

Mooncake 支持协议降级。如果 RDMA 不可用，TENT 可以自动切换到 TCP。手动配置：
- 在 Transfer Engine 初始化时指定 `protocol="tcp"` 作为 fallback
- 或者使用 TENT (`MC_USE_TENT=1`) 启用自动降级

### Q: KV cache 命中率低怎么办？

排查步骤：
1. 确认 Conductor 是否在运行，且与 Store 的 ZMQ 连接正常
2. 检查 EventManager 是否收到了 Store 的 Put/Evict 事件
3. 观察缓存命中率指标（按 token 位置分组）
4. 常见原因：
   - Evict 事件到达延迟 → Conductor 返回过时索引
   - 多轮对话超 soft TTL → 数据已被驱逐
   - PD 分离中 Prefill 和 Decode 节点不同 → Transfer Engine 传输延迟
5. 如果前缀缓存命中率确实低，考虑调用 `/advanced-optimize Mooncake "提升 Conductor 前缀缓存命中率"`

### Q: Mooncake 的日志在哪里？怎么调日志级别？

日志输出到 stderr/stdout。环境变量控制：
- `MC_LOG_LEVEL`: trace / debug / info / warn / error
- `MC_LOG_FILE`: 指定日志文件路径（可选）

建议生产环境用 `MC_LOG_LEVEL=info`，排错时临时切到 `debug`。

---

## 性能调优

### Q: 如何对 Mooncake 做性能基准测试 (Benchmark)？

Mooncake 自带 benchmark 工具：

```bash
# Transfer Engine benchmark
transfer_engine_bench \
  --local_server_name="node1" \
  --metadata_server="http://master:8080" \
  --protocol="rdma" \
  --payload_size=1048576 \  # 1MB
  --batch_size=16 \
  --iterations=1000

# Store benchmark
store_bench \
  --local_addr="node1:50051" \
  --metadata_url="http://master:8080" \
  --key_count=1000 \
  --value_size=1048576

# TENT benchmark (如果启用)
tent_bench --config=tent_config.yaml
```

### Q: Mooncake 的关键性能指标有哪些？

**Transfer Engine**：
- submit → complete 延迟 (p50/p99/p999)，按协议、按消息大小分组
- 带宽利用率 (实际 / 理论)
- Memory pin 延迟和命中率

**Store**：
- put/get 端到端延迟 (p50/p99)
- G1/G2/G3 各自命中率
- 驱逐速率（TTL 驱逐 vs. 容量驱逐）
- Master 请求延迟和 QPS

**Conductor**：
- 前缀缓存命中率（按 token 位置分组）
- 查询延迟
- 事件处理延迟和积压量

### Q: 如何提升 Mooncake Store 的 put/get 性能？

1. **零拷贝优先**：用 `put_from()` / `get_into()` 替代 `put()` / `get()`，避免数据拷贝
2. **批量操作**：用 `batch_put_from()` / `batch_get_into()` 替代逐条操作
3. **批量大小自适应**：太小 → overhead 高，太大 → 单条延迟高。建议从 16-32 开始调。
4. **NUMA 感知**：确保 client 线程在 NIC 所在 NUMA node 上运行
5. **GDR**：GPU 场景确认 GPUDirect RDMA 生效
6. **避免 page cache 污染**：G3 (SSD) 使用 Direct I/O

如果还有瓶颈，调用 `/advanced-optimize Mooncake "提升 Store put/get 性能"` 进行深度分析。

### Q: 生产环境推荐的部署拓扑是什么样的？

```
┌──────────────────────────────────────────────────────┐
│               Metadata Layer                          │
│  ETCD/Redis Cluster (3-5 nodes) + Master (2-3 nodes) │
├──────────────────────────────────────────────────────┤
│               Prefill Cluster (X nodes)               │
│  8×A100/H100 + InfiniBand/RoCE + Local NVMe SSD      │
│  运行: vLLM/SGLang + Mooncake Transfer Engine         │
├──────────────────────────────────────────────────────┤
│               Decode Cluster (Y nodes)                │
│  8×A100/H100 + InfiniBand/RoCE + 大容量 DRAM          │
│  运行: vLLM/SGLang + Mooncake Transfer Engine         │
├──────────────────────────────────────────────────────┤
│               Storage Nodes (可选)                     │
│  CPU-only, 大容量 DRAM + 多 NVMe SSD                  │
│  运行: Mooncake Client (DummyClient 模式)              │
├──────────────────────────────────────────────────────┤
│               Conductor (1-3 nodes)                   │
│  KV Cache Indexer, 前缀缓存路由                        │
└──────────────────────────────────────────────────────┘

X:Y 比例取决于 prompt:generation 长度比。典型 1:1 到 1:3。
网络: InfiniBand/RoCE 200Gbps+
```

### Q: dpbench 或 nm et  nm-san 是什么意思？ 

当前 Q&A 库未覆盖此问题。这些可能是 Mooncake 内部或 Moonshot AI 内部使用的 benchmark 代码名。建议查阅 Mooncake 源码中的 `mooncake-transfer-engine/tent/benchmark/` 或 `mooncake-store/tools/store_bench.cpp`。
