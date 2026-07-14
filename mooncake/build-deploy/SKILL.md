# Mooncake Build & Deploy Optimization

Mooncake 的构建系统、依赖管理、CI/CD 流水线和发布流程优化。

## 源代码地图

参见 `mooncake/repo-map.md` § Build & Deploy。

### 分析入口点

1. **顶级 CMake**: `CMakeLists.txt` — 构建选项和子项目编排
2. **公共 CMake 配置**: `mooncake-common/common.cmake` — 所有 USE_/WITH_ 编译选项
3. **依赖安装**: `dependencies.sh` — 自动化依赖安装脚本
4. **Wheel 打包**: `scripts/build_wheel.sh` — Python wheel 构建
5. **CI 配置**: `.github/workflows/` — GitHub Actions 流水线
6. **子模块**: `extern/yalantinglibs/`, `extern/pybind11/`

## 优化维度

### 1. CMake 构建并行化
- **代码入口**: `CMakeLists.txt`, `common.cmake`
- **当前状态**: CMake 管理多子项目
- **关键问题**:
  - 子项目之间的依赖关系是否支持并行编译？
  - 增量编译的效率？（header 依赖管理）
  - precompiled headers 的使用？
- **Domain Knowledge 映射**:
  - `performance/system-tuning/KNOWLEDGE.md` — 编译优化
  - `common/checklists/` — 构建检查清单

### 2. 依赖管理优化
- **代码入口**: `dependencies.sh`, `extern/`
- **当前状态**: shell 脚本安装依赖 + git submodules
- **关键问题**:
  - 依赖版本是否锁定？（可重现构建？）
  - 网络依赖下载的失败重试和镜像？
  - 是否可以使用系统包管理器替代源码编译？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 依赖管理
  - `common/checklists/` — 环境配置

### 3. Wheel 打包优化
- **代码入口**: `scripts/build_wheel.sh`, `mooncake-wheel/pyproject.toml`
- **当前状态**: 区分 CUDA/非 CUDA 版本
- **关键问题**:
  - Wheel 大小是否可优化？（strip symbols, 按需打包 transport）
  - 是否支持 manylinux 兼容性？
  - 不同 GPU 厂商的 wheel 变体管理？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — 发布管理

### 4. CI/CD 流水线优化
- **代码入口**: `.github/workflows/`
- **当前状态**: GitHub Actions 流水线
- **关键问题**:
  - CI 运行时间瓶颈在哪里？
  - 是否有缓存机制？（ccache, pip cache, docker layer cache）
  - 是否覆盖多平台/多 GPU 厂商的测试？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md` — CI/CD 最佳实践
  - `common/checklists/` — CI 配置检查清单

### 5. 容器化部署
- **当前状态**: 项目中是否有 Dockerfile？需验证
- **关键问题**:
  - 镜像大小（多阶段构建？基础镜像选择？）
  - GPU 驱动和 RDMA 库的容器内支持
  - Kubernetes Operator / Helm Chart？
- **Domain Knowledge 映射**:
  - `operations/cloud-infrastructure/KNOWLEDGE.md`
  - `operations/container-k8s/` (如有)

## 分析工作流

1. 阅读 CMakeLists.txt 和 common.cmake
2. 分析 CI 流水线的运行日志（如有）
3. 检查依赖安装脚本
4. 检索相关领域知识
5. 生成优化方案

## 已知优化目标

参见 `KNOWLEDGE.md`。
