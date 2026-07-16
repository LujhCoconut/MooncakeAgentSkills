# Mooncake Quick Q&A — 知识库

> **最后更新**: 2026-07-16
> **覆盖范围**: 概念架构、配置参数大全、Transfer Engine、Store、故障排查诊断树、性能调优

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

### Q: TENT 和 Transfer Engine 有什么区别？

- **Transfer Engine (TE)**：经典版本，提供多协议的基础传输能力。配置驱动，手动选择协议和参数。
- **TENT (Transfer Engine NEXT)**：下一代运行时。新增动态传输协议选择（根据实时遥测自动选协议）、多路径切片调度（大数据切成多个 slice 并行传输）、遥测驱动的闭环控制。通过 `MC_USE_TENT=1` 启用。

类比：TE = 手动挡（高性能但需要懂参数），TENT = 自动挡（自适应 + 智能调度）。

### Q: 什么是 Conductor？为什么需要它？

Conductor 是 KV Cache 的前缀感知索引器。它的工作：

1. 订阅 Store 的 Put/Evict 事件
2. 用 XXH3-64 滚动哈希建立前缀缓存索引
3. 当新请求到来时，查询 "这个 prompt 的前缀在集群里有没有已缓存的 KV cache？"
4. 如果命中 → 跳过 Prefill 计算，直接加载 KV cache（节省 30-70% Prefill 延迟）

没有 Conductor 的话，即使 KV cache 存在也无法知道，只能每次都全量 Prefill。

---

## 配置参数大全

### Transfer Engine 配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `MC_METADATA_SERVER` | string | — | metadata server 地址，格式 `http://host:port` |
| `MC_LOCAL_SERVER_NAME` | string | hostname | 本节点在集群中的标识名 |
| `MC_PROTOCOL` | string | `rdma` | 传输协议: `rdma`, `tcp`, `nvlink`, `auto` |
| `MC_DEVICE_NAME` | string | auto-detect | RDMA 设备名，如 `mlx5_0` |
| `MC_USE_TENT` | 0/1 | 0 | 启用 TENT 运行时 |
| `MC_TENT_CONFIG` | string | — | TENT 配置文件路径 (YAML) |
| `MC_BATCH_SIZE` | int | 16 | 默认批量传输大小 |
| `MC_MAX_INLINE` | int | 256 | RDMA inline 阈值 (bytes) |
| `MC_QP_COUNT` | int | 1 | RDMA Queue Pair 数量 |
| `MC_CQ_POLL_BATCH` | int | 8 | CQ polling 批量大小 |
| `MC_MEMORY_PIN_CACHE_SIZE` | int | 64 | Memory pin 缓存条目数 |
| `MC_USE_GDR` | 0/1 | auto | GPUDirect RDMA 开关 |

### Store 配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `MC_STORE_LOCAL_ADDR` | string | — | 本节点 Store 地址 `host:port` |
| `MC_STORE_METADATA_URL` | string | — | metadata 后端地址 |
| `MC_STORE_SEGMENT_SIZE` | bytes | 256MB | 每 segment 大小 |
| `MC_STORE_HARD_TTL` | seconds | 5 | 硬租约超时，过期可被驱逐 |
| `MC_STORE_SOFT_TTL` | seconds | 1800 | 软租约超时，可读但优先驱逐 |
| `MC_STORE_G1_CAPACITY` | bytes | auto | G1 (GPU HBM) 容量上限 |
| `MC_STORE_G2_CAPACITY` | bytes | auto | G2 (CPU DRAM) 容量上限 |
| `MC_STORE_G3_CAPACITY` | bytes | auto | G3 (NVMe SSD) 容量上限 |
| `MC_STORE_REPLICA_COUNT` | int | 1 | 默认副本数 |
| `MC_STORE_EVICTION_POLICY` | string | `ttl` | 驱逐策略: `ttl`, `lru`, `ttl+lru` |
| `MC_STORE_CLIENT_CONN_POOL` | int | 4 | 客户端连接池大小 |
| `MC_STORE_BATCH_MAX_SIZE` | int | 64 | 批量操作最大条目数 |

### Master 配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `MC_MASTER_ADDR` | string | — | Master 监听地址 |
| `MC_MASTER_METADATA_BACKEND` | string | `http` | 元数据后端: `http`, `etcd`, `redis` |
| `MC_MASTER_SHARD_COUNT` | int | 1024 | 分片数 |
| `MC_MASTER_LEASE_INTERVAL` | seconds | 1 | 租约续期间隔 |
| `MC_MASTER_SEGMENT_PREALLOC` | int | 0 | segment 预分配数量 |
| `MC_MASTER_EVICTION_CHECK_INTERVAL` | seconds | 5 | 驱逐检查间隔 |

### Conductor 配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `MC_CONDUCTOR_ADDR` | string | — | Conductor HTTP 监听地址 |
| `MC_CONDUCTOR_ZMQ_ADDR` | string | — | ZMQ 事件订阅地址 (连 Store) |
| `MC_CONDUCTOR_INDEX_TTL` | seconds | 3600 | 索引条目 TTL |
| `MC_CONDUCTOR_HASH_ALGO` | string | `xxh3_64` | 哈希算法 |
| `MC_CONDUCTOR_EVENT_BATCH_SIZE` | int | 32 | 事件批量处理大小 |

### 日志与调试

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `MC_LOG_LEVEL` | string | `info` | 日志级别: `trace`, `debug`, `info`, `warn`, `error` |
| `MC_LOG_FILE` | string | — | 日志文件路径，空 = stderr |
| `MC_LOG_FORMAT` | string | `text` | 日志格式: `text`, `json` |
| `MC_ENABLE_METRICS` | 0/1 | 0 | 启用 Prometheus metrics 导出 |
| `MC_METRICS_PORT` | int | 9090 | Metrics HTTP 端口 |

### RDMA 底层参数（高级）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `MC_RDMA_MAX_SEND_WR` | int | 4096 | 最大发送 Work Request 数 |
| `MC_RDMA_MAX_RECV_WR` | int | 4096 | 最大接收 Work Request 数 |
| `MC_RDMA_MAX_CQE` | int | 4096 | CQ 最大 Completion Entry 数 |
| `MC_RDMA_MTU` | int | 4096 | RDMA MTU: 1024/2048/4096 |
| `MC_RDMA_GID_INDEX` | int | 0 | RoCE GID 表索引 |
| `MC_RDMA_TRAFFIC_CLASS` | int | 0 | DSCP/流量类别 |
| `MC_RDMA_SL` | int | 0 | Service Level |
| `MC_RDMA_TIMEOUT` | int | 14 | QP 超时 (4.096µs × 2^timeout) |
| `MC_RDMA_RETRY_CNT` | int | 7 | QP 重试次数 |

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

通过 `cuda_alike.h` 统一抽象，支持：NVIDIA (CUDA, `USE_CUDA=ON`)、AMD (ROCm/HIP, `USE_HIP=ON`)、华为 Ascend (`USE_ASCEND=ON`)、Moore Threads MUSA (`USE_MUSA=ON`)、Cambricon MLU (`USE_MLU=ON`)、MetaX MACA (`USE_MACA=ON`)。

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
| AMD GPU | `-DUSE_HIP=ON` |
| 用 ETCD 做 metadata | `-DUSE_ETCD=ON` |
| 完整功能 | `-DUSE_CUDA=ON -DWITH_STORE=ON -DUSE_HTTP=ON -DUSE_REDIS=ON -DUSE_ETCD=ON -DUSE_TENT=ON` |

---

## Transfer Engine / RDMA

### Q: Transfer Engine 支持哪些传输协议？各自的延迟和带宽？

| 协议 | 适用场景 | 延迟 | 带宽 | 前提条件 |
|------|---------|------|------|---------|
| **RDMA** | 跨节点，低延迟，大数据 | 1-5µs | 100-400 Gbps | InfiniBand/RoCE 网卡 |
| **TCP** | 通用，fallback | 10-50µs | 受内核栈限制 | 无 |
| **NVLink** | 同节点 GPU↔GPU | 0.5-1µs | 100-900 GB/s | NVSwitch/NVLink bridge |
| **EFA** | AWS 环境 | 5-10µs | 100-400 Gbps | AWS EFA adapter |
| **CXL** | 内存池共享 | 100ns-1µs | 32-64 GB/s | CXL 设备 |
| **SHM** | 同节点进程间 | 0.1-0.5µs | 内存带宽 | 同一节点 |
| **NVMe-oF** | 存储分离 | 10-100µs | 受 NVMe 限制 | NVMe-oF target |

### Q: GPUDirect RDMA 怎么确认是否生效？

```bash
# 1. 检查 GPU-NIC 拓扑
transfer_engine_topology_dump

# 2. 确认 GPU 和 NIC 在同一 PCIe root complex
nvidia-smi topo -m | grep mlx5

# 3. 确认 BAR1 大小足够
nvidia-smi -q | grep -A2 "BAR1"

# 4. 运行时验证 — 用 nsys/ncu 检查是否有 D2H copy
# 如果有 Device to Host 传输 → GDR 未生效
```

必要条件：GPU 和 NIC 在同一 PCIe root complex 下，且支持 ACS redirect（或已禁用）。跨 NUMA 的 GPU-NIC 通常不支持 GDR。

---

## 故障排查诊断树

### 诊断树 1: Transfer Engine 无法初始化

```
TE::init() 失败
├─ 错误: "Cannot connect to metadata server"
│  ├─ curl http://<metadata_server>/health → 不通?
│  │  └─ 检查: 防火墙、地址格式、服务是否启动
│  └─ curl 通但 init 仍失败?
│     └─ 检查: metadata_server 的协议头 (http:// vs etcd://)
│
├─ 错误: "No RDMA device found"
│  ├─ ibv_devinfo → 有输出?
│  │  ├─ 无 → RDMA 驱动未安装 / 硬件不支持
│  │  └─ 有 → 继续
│  ├─ ibstat → 状态 = ACTIVE?
│  │  └─ 非 ACTIVE → 检查网线、交换机、SM (InfiniBand)
│  └─ ulimit -l → 是否 unlimited?
│     └─ 不是 → sudo setcap cap_ipc_lock+ep <binary> 或 ulimit -l unlimited
│
├─ 错误: "Cannot register memory"
│  ├─ 是否超过 max locked memory?
│  │  └─ cat /proc/self/limits | grep "locked memory"
│  └─ GPU memory? 检查 nvidia-smi 确认 BAR1 未满
│
└─ 错误: "TENT config parse error"
   └─ 检查 MC_TENT_CONFIG 路径、YAML 格式、schema
```

### 诊断树 2: Store put/get 操作失败或延迟异常

```
put()/get() 异常
├─ put() 失败: "No segment available"
│  ├─ Master 是否在线? → 检查 Master 进程
│  ├─ 所有节点的 G2/G3 是否已满?
│  │  └─ 检查: 驱逐是否正常工作? TTL 配置是否合理?
│  └─ segment_manager 是否有 bug? → 查 Master 日志
│
├─ get() 返回 "Key not found"
│  ├─ 对象是否已过期 (TTL)?
│  │  ├─ 检查: key 的创建时间和当前 hard/soft TTL
│  │  └─ 如果 multi-turn 对话中丢失 → 增加 soft TTL
│  ├─ 对象是否被驱逐?
│  │  └─ 检查: Master 的 eviction 日志
│  └─ Conductor 返回了 stale index?
│     └─ 检查: Evict 事件到达 Conductor 的延迟
│
├─ put/get 延迟异常高 (>10× baseline)
│  ├─ RDMA 链路是否降级?
│  │  └─ ibstat → 检查 link speed
│  ├─ 是否走 TCP fallback 而非 RDMA?
│  │  └─ 检查日志中的 transport 类型
│  ├─ NUMA 亲和性是否正确?
│  │  └─ transfer_engine_topology_dump → 检查 GPU-NIC 距离
│  └─ GDR 是否失效?
│     └─ nvidia-smi topo -m → 确认 PIX (非 PHB/SYS)
│
└─ 批量操作比单条操作还慢
   ├─ batch size 是否过大? → 减小到 16-32
   ├─ 是否在 batch 内部做串行化? → 检查 batch scheduler 日志
   └─ CQ polling 是否 batch 不够? → 增大 MC_CQ_POLL_BATCH
```

### 诊断树 3: 推理服务端到端延迟异常

```
端到端推理延迟升高
├─ Prefill 阶段变慢
│  ├─ 前缀缓存命中率下降?
│  │  ├─ 检查 Conductor 的命中率指标
│  │  ├─ Conductor 是否在线? ZMQ 连接是否正常?
│  │  └─ Store Evict 事件是否到达 Conductor?
│  ├─ 是否走了远程 Prefill (原本应该是本地的)?
│  │  └─ 检查 PD 分离的调度策略
│  └─ Prefill→Decode 的 KV cache 传输延迟?
│     └─ 检查 TE transport latency, 是否走对了协议?
│
├─ Decode 阶段变慢
│  ├─ KV cache 读取延迟高?
│  │  ├─ 是否每次 decode step 都从 Store get? (应该是 cache 在 local)
│  │  └─ 检查 G1 命中率 — 如果 <90% 则 G1 太小或驱逐太快
│  ├─ GPU memory 不足导致 swap?
│  │  └─ nvidia-smi → 检查 used/total memory
│  └─ Batch size 是否不匹配? (P refill 和 Decode 的 batch 不同)
│
└─ 整体吞吐下降
   ├─ Master 是否成为瓶颈?
   │  └─ 检查 Master QPS 和延迟
   ├─ 存储节点负载是否均衡?
   │  └─ 检查各节点的 segment 利用率分布
   └─ 网络拥塞?
      └─ 检查 RDMA CQ depth, retry count, 交换机端口统计
```

### 诊断树 4: 集群节点异常

```
节点异常
├─ 节点失联 (心跳超时)
│  ├─ 网络不可达? → ping, traceroute
│  ├─ RDMA 链路断开? → ibstat, 检查交换机
│  ├─ 进程 OOM? → dmesg | grep -i oom
│  └─ GPU 掉卡? → nvidia-smi 是否可见
│
├─ 节点 CPU 100%
│  ├─ 哪个进程? → top / htop
│  ├─ CQ polling 忙等? → 检查 MC_CQ_POLL_BATCH, 是否使用了 event-driven
│  ├─ pybind11 GIL 竞争? → py-spy 或 Python profiler
│  └─ 压缩/加密占用? → 检查是否开启传输压缩
│
├─ 节点内存异常
│  ├─ Memory pin 过多? → cat /proc/meminfo | grep Unevictable
│  ├─ Page cache 膨胀 (G3 用 buffered I/O)? → 切 Direct I/O
│  └─ Memory leak? → valgrind / heaptrack / pmap
│
└─ 节点 SSD 异常
   ├─ 延迟飙升? → iostat -x 1, 检查 await
   ├─ 寿命耗尽? → nvme smart-log /dev/nvme0
   └─ 写入放大过高? → 检查对齐写入、Direct I/O 状态
```

### 常见错误速查表

| 错误信息 | 根因 | 修复 |
|---------|------|------|
| `Cannot create QP` | RDMA QP 资源不足 | 增大 `MC_RDMA_MAX_SEND_WR`、检查 `ulimit -l` |
| `QP transition to ERROR state` | RDMA 链路故障 | 检查网线/交换机、`ibstat` 查看链路状态 |
| `Retry count exceeded` | RDMA 重传耗尽 | 增大 `MC_RDMA_RETRY_CNT`，或排查丢包根因 |
| `Completion with error (WC status: 5)` | RDMA remote access error | 远端内存可能被 unpin 或 rkey 失效 |
| `Metadata server timeout` | Master 或 metadata backend 过载 | 加 ETCD/Redis 节点，或给 metadata 加缓存 |
| `Segment allocation timeout` | Master 无可用 segment | 增加节点或 segment、降低 TTL |
| `Lease renewal failed` | 客户端与 Master 断连 | 检查网络、确认 Master 进程 |
| `Memory registration failed` | 超过 locked memory limit | `ulimit -l unlimited` |
| `BAR1 exhausted` | GPU BAR1 空间不足 (pin 太多) | 减少并发 pin 数量、增大 BAR1 |
| `ZMQ connection lost` | Conductor 与 Store 断连 | 检查 ZMQ 端口、重启 Conductor |
| `C++ coroutine allocation failed` | 协程栈/内存不足 | 检查 `-fcoroutines` 编译选项、堆大小 |

---

## 性能调优

### Q: 如何做性能基准测试？

```bash
# Transfer Engine benchmark
transfer_engine_bench \
  --local_server_name="node1" \
  --metadata_server="http://master:8080" \
  --protocol="rdma" \
  --payload_size=1048576 \
  --batch_size=16 \
  --iterations=1000

# Store benchmark
store_bench \
  --local_addr="node1:50051" \
  --metadata_url="http://master:8080" \
  --key_count=1000 --value_size=1048576

# TENT benchmark
tent_bench --config=tent_config.yaml
```

### Q: 生产环境推荐部署拓扑？

```
Metadata Layer: ETCD/Redis Cluster (3-5 nodes) + Master (2-3 nodes)
Prefill Cluster: 8×A100/H100 + IB/RoCE + NVMe SSD (X nodes)
Decode Cluster:  8×A100/H100 + IB/RoCE + 大 DRAM (Y nodes, X:Y = 1:1~1:3)
Storage Nodes:   CPU-only, 大 DRAM + 多 NVMe (可选)
Conductor:       1-3 nodes for KV index
网络: IB/RoCE 200Gbps+
```

### Q: 如何定位当前系统的瓶颈？考虑使用排队论分析。

Mooncake 中每个组件都有隐含的队列——RDMA send queue、Store request queue、Master allocation queue、Conductor event queue 等。瓶颈通常是队列最长（利用率最高）的那个环节。

如需系统性地用排队论建模分析端到端延迟，调用：
```bash
/advanced-optimize "对 Mooncake 系统进行排队论建模分析"
```

这会触发 `mooncake/queueing-theory/SKILL.md` 中的方法论：识别所有排队点 → 选择模型 (M/M/1, M/M/c, M/G/1…) → 计算平均等待时间 → 定位瓶颈 → 给出优化建议。
