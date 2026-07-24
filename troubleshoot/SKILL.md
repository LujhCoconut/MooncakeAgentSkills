---
name: troubleshoot
description: Mooncake 部署与运行时故障系统化诊断。检查服务状态 (mooncake_master, metadata server)、RDMA 设备、环境变量、网络连通性、内存限制、日志分析。触发词："troubleshoot"、"debug"、"diagnose"、"fix"、"排查"、"报错"、"启动失败"、"连接不上"、错误码如 "Error from etcd client"、"No matched device found"、"Failed to register memory"、"NO_AVAILABLE_HANDLE"。
argument-hint: "[--quick|<错误信息>]"
---

# Mooncake 部署故障排查

你是 Mooncake 部署故障排查专家。你的任务是**系统化诊断问题**并给出可操作的解决方案。

## 关联子命令

| 场景 | 使用 |
|------|------|
| 排查完需要深入了解源码实现 | `/mooncake-agent-skills optimize "..."` |
| 概念/配置类问题（非故障） | `/mooncake-agent-skills qa "..."` |
| API 用法问题（非故障） | `/mooncake-agent-skills api-reference "..."` |

---

## 诊断策略

**从简到深**：先做简单检查（服务、连通性），再深入复杂问题（RDMA、内存注册）。逐项报告，不要一次性 dump。

---

## 1. 服务状态检查

检查关键服务是否运行：

```bash
# Check mooncake_master
ps aux | grep mooncake_master | grep -v grep

# Check port usage
netstat -tuln | grep -E '(50051|8080|2379|9003)'

# If using etcd
ps aux | grep etcd | grep -v grep
```

**常见问题：**
- `bind address already in use` → 端口冲突，用 `--rpc_port` 换端口
- Master 未运行 → 检查启动日志

---

## 2. Metadata Server 连通性

Metadata server 是节点发现和协调的关键。

```bash
# Test etcd connectivity
curl -s http://127.0.0.1:2379/version

# Or test custom metadata server
curl -s $MC_METADATA_SERVER

# Check for proxy interference
echo "http_proxy: $http_proxy"
echo "https_proxy: $https_proxy"
```

**常见问题：**
- `Error from etcd client` → Metadata server 不可达
  - **Fix:** 确保 etcd 绑定到 `0.0.0.0` 而非 `127.0.0.1`：
    ```bash
    etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://<your_ip>:2379
    ```
  - **Fix:** 禁用 HTTP 代理：
    ```bash
    unset http_proxy https_proxy
    ```

---

## 3. 环境变量检查

验证关键环境变量设置正确：

```bash
# Display all MC_* variables
env | grep ^MC_

# Key variables to check:
echo "MC_METADATA_SERVER: $MC_METADATA_SERVER"
echo "MC_FORCE_TCP: $MC_FORCE_TCP"
echo "MC_LOG_LEVEL: $MC_LOG_LEVEL"
echo "MC_YLT_LOG_LEVEL: $MC_YLT_LOG_LEVEL"
echo "MC_MS_AUTO_DISC: $MC_MS_AUTO_DISC"
echo "MC_MS_FILTERS: $MC_MS_FILTERS"
echo "MC_GID_INDEX: $MC_GID_INDEX"
echo "MC_MTU: $MC_MTU"
echo "MC_IB_PORT: $MC_IB_PORT"
echo "MC_ENABLE_DEST_DEVICE_AFFINITY: $MC_ENABLE_DEST_DEVICE_AFFINITY"
```

> 完整环境变量列表和默认值见 `/mooncake-agent-skills qa "MC_ 环境变量"` 或 `qa/KNOWLEDGE.md` 配置参数表（49 项）。

---

## 4. RDMA 设备检查

仅在启用 RDMA 时执行（`MC_FORCE_TCP=true` 时跳过）：

```bash
# List RDMA devices
ibv_devices

# Check device details and status
ibv_devinfo

# Check for ACTIVE ports
ibv_devinfo | grep -A 10 "state:"

# Check GID addresses (should NOT be all zeros)
ibv_devinfo | grep -A 20 "GID"

# Check peer memory modules
lsmod | grep peer_mem
lsmod | grep nvidia_peer_mem

# Check QP count (if "Failed to create QP" error)
rdma resource show qp
```

**常见问题：**
- `No matched device found` → RDMA 设备名不存在
  - **Fix:** 用 `ibv_devices` 获取正确设备名
- `Device XXX port not active` → RDMA 端口非 ACTIVE 状态
  - **Fix:** 检查物理线缆连接，`ibv_devinfo | grep state`
  - **Fix:** 尝试不同端口：设置 `MC_IB_PORT`
- GID 全零 → 错误的 GID index
  - **Fix:** 设置 `MC_GID_INDEX=1`（或 2、3，取决于网络）
- `Failed to create QP: Cannot allocate memory` → 创建了过多 QP
  - **Fix:** 设置 `MC_ENABLE_DEST_DEVICE_AFFINITY=1`

> **RDMA 排查深度分析**：如果想理解 RDMA QP 创建机制和优化策略 → `/mooncake-agent-skills optimize "RDMA QP 管理和拓扑亲和性"`

---

## 5. 内存和资源限制

检查影响 RDMA 内存注册的系统限制：

```bash
# Check ulimits
ulimit -a

# Focus on max locked memory
ulimit -l

# Check RDMA device memory limits
ibv_devinfo -v | grep max_mr_size

# Check dmesg for memory errors
dmesg -T | tail -50 | grep -i "out of mr size"
```

**常见问题：**
- `Failed to register memory: Input/output error` → 内存注册超限
  - **诊断：** `ibv_devinfo -v | grep max_mr_size`
  - **Fix:** 减小内存分配或拆分
- Cannot allocate memory → ulimit 限制
  - **Fix:** 设置 unlimited locked memory：
    ```bash
    ulimit -l unlimited
    ```
  - **永久修：** `/etc/security/limits.conf`:
    ```
    * soft memlock unlimited
    * hard memlock unlimited
    ```

---

## 6. 网络连通性

测试节点间连通性：

```bash
# Test basic RDMA connectivity
ib_write_bw -d <device_name> -R

# On peer node:
ib_write_bw -d <device_name> -R <server_ip>

# Test GPU Direct RDMA (if CUDA enabled)
ib_write_bw -d <device_name> -R -x gdr

# On peer node:
ib_write_bw -d <device_name> -R -x gdr <server_ip>

# Test DNS resolution
nslookup <connectable_name>
ping <connectable_name>
```

**常见问题：**
- `connection refused` → `connectable_name` 或 `rpc_port` 不正确
  - **Fix:** 确保 `connectable_name` 不是 loopback（127.0.0.1/localhost）
  - **Fix:** 使用实际 LAN/WAN IP 或有效 hostname
- `Failed to exchange handshake` → RDMA 连接建立失败
  - **Fix:** 验证 MTU 匹配：设置 `MC_MTU`
  - **Fix:** 验证 GID 有效（非全零）
  - **Fix:** 先用 `ib_send_bw` 在节点间测试

---

## 7. 日志分析

搜索日志中的常见错误模式：

### Metadata/Connectivity Errors
| 错误信息 | 含义 | 相关 QA |
|---------|------|---------|
| `Error from etcd client` | 无法连接 metadata server | `qa` → "etcd 报错怎么办" |
| `ERR_METADATA` | Metadata server 通信失败 | `qa` → "metadata server" |
| `ERR_DNS` | `local_server_name` 无效 | - |

### RDMA Errors
| 错误信息 | 含义 | 源码定位 |
|---------|------|---------|
| `No matched device found` | RDMA 设备名不存在 | `mooncake-transfer-engine/src/transfer_engine.cpp` → device init |
| `Device XXX port not active` | RDMA 端口非 ACTIVE | 检查网卡物理状态 |
| `Failed to exchange handshake description` | RDMA 握手失败 | `mooncake-transfer-engine/src/transport/rdma/` |
| `Failed to modify QP to RTR` | MTU/GID 不匹配 | `mooncake-transfer-engine/src/transport/rdma/rdma_context.cpp` |
| `Failed to register memory` | 内存注册超限 | `mooncake-transfer-engine/src/transfer_engine.cpp` → `register_memory()` |
| `Failed to create QP` | QP 过多 | 设置 `MC_ENABLE_DEST_DEVICE_AFFINITY=1` |
| `Worker: Process failed for slice` | 网络不稳定 | TENT 切片执行失败 |
| `work request flushed error` | 级联错误（找第一条错误） | - |

### Store Errors
| 错误码 | 名称 | 含义 | 源码定位 |
|--------|------|------|---------|
| -200 | `NO_AVAILABLE_HANDLE` | 内存池耗尽 | `mooncake-store/src/allocator/` |
| -707 | `LEASE_EXPIRED` | 传输期间 lease 过期 | `mooncake-store/src/master/` → lease mgmt |
| -704 | `OBJECT_NOT_FOUND` | 对象不存在 | `mooncake-store/src/client.cpp` |
| -101 | `SEGMENT_NOT_FOUND` | 无可用 segment | `mooncake-store/src/master/` → segment mgmt |
| -900 | `RPC_FAIL` | RPC 失败 | `mooncake-store/src/rpc/` |
| -1000 | `ETCD_OPERATION_ERROR` | etcd 操作失败 | `mooncake-store/src/metadata/` |

> **Store 错误深度排查**：`NO_AVAILABLE_HANDLE` 和驱逐策略相关 → `/mooncake-agent-skills optimize "Store 内存池和驱逐策略优化"`

### Port/Service Errors
| 错误信息 | 修复 |
|---------|------|
| `bind address already in use` | 换端口：`--rpc_port=50052` |
| `Failed to get description of XXX` | 确保 segment name 匹配 peer 的 `local_hostname` |

---

## 8. 配置验证

验证配置正确性：

```bash
# Check connectable_name is not loopback
hostname -I

# Verify master startup flags
ps aux | grep mooncake_master

# Check if using correct protocol
env | grep MC_FORCE_TCP
```

**关键检查项：**
- `connectable_name` 必须是非 loopback IP 或有效 hostname
- MTU 和 GID 配置必须匹配网络环境
- RDMA 设备名必须在机器上存在
- 端口不能已被其他服务占用

---

## 错误码速查

### Transfer Engine 错误码

| Code | Name | Meaning | Fix |
|------|------|---------|-----|
| 0 | Success | 正常 | - |
| -12 | ERR_ADDRESS_NOT_REGISTERED | 内存未注册 | 使用前先 `register_memory()` |
| -14 | ERR_DEVICE_NOT_FOUND | RDMA 设备未找到 | `ibv_devices` 检查设备名 |
| -16 | ERR_DNS | 无效 local_server_name | 使用有效 IP/hostname |
| -19 | ERR_REJECT_HANDSHAKE | 对端拒绝握手 | 检查对端日志 |
| -20 | ERR_METADATA | Metadata server 不可达 | 检查 etcd/HTTP server |

### Store 错误码

| Code | Name | Meaning | Fix |
|------|------|---------|-----|
| 0 | Success | 操作成功 | - |
| -200 | NO_AVAILABLE_HANDLE | 内存池耗尽 | 增大 segment size |
| -707 | LEASE_EXPIRED | Lease 过期 | 增大 lease TTL |
| -704 | OBJECT_NOT_FOUND | 对象不存在 | 检查 key |
| -101 | SEGMENT_NOT_FOUND | 无可用 segment | 检查 segment 注册 |
| -900 | RPC_FAIL | RPC 失败 | 检查网络/master |
| -1000 | ETCD_OPERATION_ERROR | etcd 操作失败 | 检查 etcd 状态 |

---

## 输出格式

提供结构化诊断报告：

```
🔍 MOONCAKE DEPLOYMENT DIAGNOSTICS
==================================

✅ PASSED CHECKS:
- Service status: mooncake_master running on port 50051
- Metadata server: etcd accessible at http://127.0.0.1:2379
- Environment: MC_METADATA_SERVER set correctly
- [other passing checks]

❌ FAILED CHECKS:
- RDMA device: mlx5_0 port not ACTIVE (state: PORT_DOWN)  [Severity: CRITICAL]
- Memory limits: max locked memory is 64KB (too low)  [Severity: HIGH]
- [other failures with specific error messages]

⚠️  WARNINGS:
- GID index not set, may cause connection issues  [Severity: MEDIUM]
- HTTP proxy variables set, may interfere with metadata server  [Severity: LOW]
- [other potential issues]

🔧 RECOMMENDED FIXES:

1. [CRITICAL] Fix RDMA port status:
   - Check physical cable connections
   - Verify driver configuration
   - Command: ibv_devinfo | grep -A 10 "state:"

2. [HIGH] Increase memory limits:
   ulimit -l unlimited
   # Or permanently in /etc/security/limits.conf:
   * soft memlock unlimited
   * hard memlock unlimited

3. [MEDIUM] Set GID index:
   export MC_GID_INDEX=1

4. [LOW] Disable HTTP proxy:
   unset http_proxy https_proxy

📋 SUMMARY:
[Brief 2-3 sentence conclusion about deployment health and next steps]

🔗 RELATED Q&A:
- qa: "RDMA QP 报错怎么办"
- qa: "Master 挂了推理还能继续吗"
```

### 严重度分类

| 级别 | 标记 | 定义 |
|------|------|------|
| CRITICAL | 🔴 | 服务完全不可用，必须立即修复 |
| HIGH | 🟠 | 核心功能受影响，强烈建议修复 |
| MEDIUM | 🟡 | 潜在风险或非最优配置 |
| LOW | 🔵 | 建议项，不影响基本功能 |

---

## 排查工作流

1. **从简开始**：先检查服务和基本连通性
2. **仔细读日志**：第一条错误通常是根因（后续错误是级联的）
3. **增量测试**：用 `MC_FORCE_TCP=true` 隔离 RDMA 问题
4. **验证基础**：检查 connectable_name、端口、环境变量
5. **使用诊断工具**：ibv_devices, ibv_devinfo, ib_write_bw, curl
6. **参考文档**：检查错误码和排查指南

---

## Quick 诊断模式

当用户只提供错误信息但未指定完整排查范围时，执行 quick 诊断：

```bash
# 一键收集关键信息
echo "=== Services ===" && ps aux | grep -E '(mooncake_master|etcd)' | grep -v grep
echo "=== Ports ===" && netstat -tuln | grep -E '(50051|8080|2379)' | head -5
echo "=== Env ===" && env | grep ^MC_ | head -10
echo "=== RDMA ===" && ibv_devices 2>/dev/null || echo "No RDMA tools"
echo "=== Memory ===" && ulimit -l
echo "=== Errors ===" && dmesg -T | tail -20 | grep -iE '(mooncake|rdma|mlx|memory)'
```

根据输出自动化判断最可能的故障类别，然后深入对应章节。

---

## 快速修复命令

**正确启动 metadata server：**
```bash
etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://<your_ip>:2379
```

**启用详细日志：**
```bash
export MC_LOG_LEVEL=0
export MC_YLT_LOG_LEVEL=debug
```

**强制 TCP 模式测试：**
```bash
export MC_FORCE_TCP=true
```

**修复内存限制：**
```bash
ulimit -l unlimited
```

**修复过多 QP：**
```bash
export MC_ENABLE_DEST_DEVICE_AFFINITY=1
```

**修复 GID 问题：**
```bash
export MC_GID_INDEX=1  # or 2, 3 depending on network
```

**使用不同端口：**
```bash
mooncake_master --rpc_port=50052
```
