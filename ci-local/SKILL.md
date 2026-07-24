---
name: ci-local
description: Mooncake 提交前本地 CI 验证。运行 scripts/run_ci_test.sh 执行预提交检查，含 quick 快速模式、变更分析、失败根因定位。触发词："提交 PR 前验证"、"run ci test"、"run local CI"、"check my branch"、"test before PR"、"pre-submit validation"、"reproduce CI locally"、"本地 CI"。
argument-hint: "[--quick|--skip-path-filter|--install-deps|--keep-services]"
---

# Mooncake 提交前本地 CI 验证

**前置条件**：必须在 Mooncake 源码仓库根目录下执行。本 skill 依赖 `scripts/run_ci_test.sh`（kvcache-ai/Mooncake 仓库自带）。

使用 `bash scripts/run_ci_test.sh` 作为默认入口。这是 PR 提交前本地验证的唯一通道，已协调 GitHub Actions 中可复现的部分。

## 触发条件

当用户要求以下任一操作时，走本 skill：
- 提交 PR 前本地验证
- run ci test / run local CI
- check my branch before PR
- reproduce CI locally
- 跑一下 CI / 本地 CI 验证

---

## 关联子命令

| 场景 | 使用 |
|------|------|
| CI 失败，需要分析构建系统瓶颈 | `/mooncake-agent-skills optimize "CMake 并行编译优化"` |
| CI 通过，准备提交 PR，需要代码审查 | `/mooncake-agent-skills review` |
| 不是 CI 问题，是 Mooncake 部署报错 | `/mooncake-agent-skills troubleshoot` |

---

## 默认入口

```bash
bash scripts/run_ci_test.sh
```

### 覆盖的阶段

- GitHub-like `paths-filter` 对 `origin/main`
- `typos` 拼写检查
- `scripts/code_format.sh --check` 代码格式
- 默认 CMake configure/build/install（`build-ci-local`）
- `ctest`
- wheel build（`build-wheel-local`）
- wheel 安装验证
- `scripts/run_tests.sh`
- 选定的 Python API 和集成测试
- 每阶段摘要 + 日志：`local_test/run-ci-logs/<timestamp>/`

---

## 标准 Agent 工作流

1. **变更分析（前置）**：运行 CI 前，先分析当前分支相对 `origin/main` 的变更范围：
   ```bash
   git diff --stat origin/main...HEAD
   ```
   报告变更文件数和类型（C++/Python/CMake/CI/Docs），帮助用户理解 CI 会覆盖哪些检查。

2. 从仓库根目录运行 `bash scripts/run_ci_test.sh`（除非用户明确要求窄范围）

3. 读取阶段摘要，不 dump 原始终端输出

4. 向用户报告：
   - ✅ passed stages
   - ❌ failed stages
   - 🚫 blocked stages（环境/依赖问题）
   - ⏭️ unsupported stages（本地不支持）
   - 是否 `paths-filter` 跳过了下游阶段
   - 日志目录 `local_test/run-ci-logs/...`

5. 如果有失败，检查对应阶段日志，总结根因

---

## 常用选项

### Quick 模式（新增）

跳过耗时的 ASan 编译 + ctest + wheel build，仅做快速门禁：

```bash
bash scripts/run_ci_test.sh --quick
```

Quick 模式覆盖：
- `paths-filter` 变更检测
- `typos` 拼写检查
- `scripts/code_format.sh --check` 代码格式
- 快速编译（Release 模式，无 ASan）

> **适用场景**：快速迭代中只想确认格式和拼写无误；完整验证请用默认模式。

### 强制全量

忽略 `paths-filter` 跳过机制，强制跑全量：

```bash
bash scripts/run_ci_test.sh --skip-path-filter
```

### 自定义 base ref

```bash
bash scripts/run_ci_test.sh --base origin/main
```

### 自动安装依赖

```bash
bash scripts/run_ci_test.sh --install-deps
```

### 保留服务用于调试

```bash
bash scripts/run_ci_test.sh --keep-services
```

---

## Minimal Example

用户 prompt：
> 提交 PR 前，帮我跑一遍本地 CI 验证当前分支。

期望动作：
```bash
bash scripts/run_ci_test.sh
```

如果用户想跳过路径过滤强制全量：
```bash
bash scripts/run_ci_test.sh --skip-path-filter
```

详见 `.claude/skills/mooncake-ci-local/examples/minimal.md`。

---

## 结果解读

| 状态 | 含义 |
|------|------|
| `passed` | 阶段在本地通过 |
| `failed` | 阶段复现了真实本地失败，需要调查 |
| `blocked` | 本地环境或依赖问题阻止执行 |
| `unsupported` | 设计上不在本地 lane 运行（需外部平台/特殊硬件/非默认构建） |

如果 `paths-filter` 跳过了下游阶段，说明当前分支仅改了非源码路径（如 docs/、.github/），无需跑完整 CI。

---

## 当前本地覆盖范围

### 默认包含

- spell check
- code format check
- 默认 ASan CMake lane（`build-ci-local`）
- `ctest`
- wheel build + 安装测试
- `scripts/run_tests.sh`
- 选定的 Python API 测试

### 设计上不支持（本地 lane）

- Ascend jobs
- T-one integration jobs
- MUSA jobs
- Docker image build jobs
- CUDA 13 wheel jobs
- PG-backend tests（不在默认 wheel build 中）
- Python drain-http API stage（本地 ASan lane）

---

## 针对性重跑（调试用）

**仅在完整脚本识别出失败区域后**，或用户明确要求窄范围时使用。

重跑特定 C++ 测试：
```bash
cd build-ci-local
MC_METADATA_SERVER=http://127.0.0.1:8080/metadata DEFAULT_KV_LEASE_TTL=500 ctest -R <pattern> --output-on-failure
```

重跑 Python wheel 集成 lane：
```bash
source test_env/bin/activate
MC_STORE_MEMCPY=false TEST_SSD_OFFLOAD_IN_EVICT=true ./scripts/run_tests.sh
```

重跑 safetensor unittest：
```bash
source test_env/bin/activate
python -m unittest mooncake-wheel.tests.test_safetensor_functions
```

---

## Agent 注意事项

- **优先用仓库脚本**，不要逐步重建 CI 工作流
- **保留 `build-ci-local` 和 `build-wheel-local` 的分离**
- 从日志中**总结**失败阶段，不要粘贴原始输出
- 如果用户只问"分支是否安全可以提 PR"，默认回答路径是 `bash scripts/run_ci_test.sh`
- **变更分析先行**：运行 CI 前先展示 `git diff --stat`，帮助用户建立变更范围的心理模型

---

## 前置条件检查脚本

如果 `scripts/run_ci_test.sh` 不存在或无法运行，执行前置条件检查：

```bash
# 确认在 Mooncake 仓库中
test -f scripts/run_ci_test.sh && echo "✓ In Mooncake repo" || echo "✗ Not in Mooncake repo"

# 确认必要工具
which cmake && which python3 && which pip3

# 检查子模块
git submodule status
```

详见 `scripts/check-prerequisites.sh`。
