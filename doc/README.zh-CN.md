**[English](../README.md)** | **[繁體中文](README.zh-TW.md)** | **简体中文** | **[日本語](README.ja.md)**

# Codex CLI Docker 环境

[![CI](https://github.com/ycpss91255-docker/codex_cli/actions/workflows/main.yaml/badge.svg)](https://github.com/ycpss91255-docker/codex_cli/actions/workflows/main.yaml) [![License](https://img.shields.io/badge/License-Apache--2.0-blue?style=flat-square)](../LICENSE)

OpenAI Codex CLI 的 Docker-in-Docker（DinD）开发容器，提供 CPU 与 NVIDIA GPU 两种版本。以非 root 用户运行，并与主机的 UID/GID 相符。

## 目录

- [TL;DR](#tldr)
- [概述](#概述)
- [前置条件](#前置条件)
- [快速开始](#快速开始)
- [对话持久化](#对话持久化)
- [执行多个实例](#执行多个实例)
- [验证方式](#验证方式)
  - [OAuth（交互式登录）](#oauth交互式登录)
  - [API 密钥（加密）](#api-密钥加密)
- [作为 Subtree 使用](#作为-subtree-使用)
- [设置](#设置)
- [Smoke Tests](#smoke-tests)
- [架构](#架构)
  - [Dockerfile 阶段](#dockerfile-阶段)
  - [Compose 服务](#compose-服务)
  - [入口点流程](#入口点流程)
  - [预安装工具](#预安装工具)
  - [容器能力](#容器能力)

## TL;DR

```bash
./build.sh && ./run.sh    # Build and run (CPU, default)
```

- 附带 OpenAI Codex CLI 的隔离 Docker-in-Docker 容器
- 非 root 用户，自动从主机检测 UID/GID
- 首次启动时自动复制 OAuth 凭证，对话记录持久化存储于本地
- 可选择以 GPG AES-256 加密 API 密钥
- 默认使用 CPU，GPU 版本请执行 `./run.sh devel-gpu`

## 概述

```mermaid
graph TB
    subgraph Host
        H_OAuth["~/.codex<br/>(OAuth 凭证)"]
        H_WS["工作区<br/>(WS_PATH)"]
        H_Data["数据目录<br/>(agent_* or ./data/)"]
    end

    subgraph "Container (DinD)"
        EP["entrypoint.sh"]
        DinD["dockerd<br/>(隔离)"]
        Codex["Codex CLI"]
        Tools["git, python3, jq,<br/>ripgrep, make, cmake..."]

        EP -->|"1. 启动"| DinD
        EP -->|"2. 复制凭证<br/>（首次运行）"| Codex
        EP -->|"3. 解密 API 密钥<br/>（如有 .env.gpg）"| Tools
    end

    H_OAuth -->|"只读挂载"| EP
    H_WS -->|"挂载<br/>~/work"| Tools
    H_Data -->|"挂载<br/>~/.codex"| Codex

```

```mermaid
graph LR
    subgraph "Dockerfile 阶段"
        sys["sys<br/>用户, 语言, 时区"]
        base["base<br/>开发工具, Docker"]
        devel["devel<br/>codex cli"]
        test["test<br/>Bats smoke test"]
    end

    sys --> base --> devel --> test

    subgraph "Compose Services"
        S_CPU["devel<br/>(CPU, default)"]
        S_GPU["devel-gpu<br/>(NVIDIA GPU)"]
        S_Test["test<br/>（临时性）"]
    end

    devel -.-> S_CPU
    devel -.-> S_GPU
    test -.-> S_Test

```

```mermaid
flowchart LR
    subgraph "run.sh"
        A["Generate .env<br/>(template)"] --> B["Derive BASE_IMAGE<br/>(post_setup.sh)"]
        B --> C{"--data-dir?"}
        C -->|yes| D["Use specified dir"]
        C -->|no| E{"agent_* found?"}
        E -->|yes| F["Use agent_* dir"]
        E -->|no| G["Use ./data/"]
        D --> H["docker compose run"]
        F --> H
        G --> H
    end
```

## 前置条件

- Docker 并支持 Compose V2
- GPU 版本需要 [nvidia-container-toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- 主机端已完成 Codex CLI 的 OAuth 登录（`codex`）

## 快速开始

```bash
# Build (auto-generates .env on every run)
./build.sh              # CPU variant (default)
./build.sh devel-gpu    # GPU variant
./build.sh --no-env test  # 构建但不更新 .env

# Run
./run.sh                          # CPU variant (default)
./run.sh devel-gpu                # GPU variant
./run.sh --data-dir ../agent_foo  # Specify data directory
./run.sh --no-env -d              # 后台启动，跳过 .env 更新

# Exec into running container
./exec.sh
```

## 对话持久化

对话记录与 Session 数据通过 挂载 持久化存储，容器重启后仍可保留。

`run.sh` 会从项目目录向上自动扫描是否存在 `agent_*` 目录。若找到，数据将存储于该目录；否则回退使用 `./data/`。

```
# Example: if ../agent_myproject/ exists
../agent_myproject/
└── .codex/     # Codex CLI conversations, settings, session

# Fallback: no agent_* directory found
./data/
└── .codex/
```

- 首次启动：OAuth 凭证会从主机复制到数据目录
- 后续启动：数据目录已有数据，直接使用（不会覆写）
- 可自由复制、备份或移动数据目录
- 手动指定：`./run.sh --data-dir /path/to/dir`

## 执行多个实例

使用 `--project-name`（`-p`）创建完全隔离的实例，每个实例各有独立的命名 volume：

```bash
# Instance 1
docker compose -p codex1 --env-file .env run --rm devel

# Instance 2 (in another terminal)
docker compose -p codex2 --env-file .env run --rm devel

# Instance 3
docker compose -p codex3 --env-file .env run --rm devel
```

若要执行多个实例，请创建各自独立的 `agent_*` 目录：

```bash
mkdir ../agent_proj1 ../agent_proj2

./run.sh --data-dir ../agent_proj1
./run.sh --data-dir ../agent_proj2
```

凭证、对话记录与 Session 数据完全隔离。清理时只需删除对应目录：

```bash
rm -rf ../agent_proj1
```

## 验证方式

支持两种方式，可同时使用。

### OAuth（交互式登录）

适用于交互式 CLI 使用。请先在主机上登录：

```bash
codex    # Log in to Codex CLI
```

凭证（`~/.codex`）以只读方式挂载至容器，并在首次启动时复制到数据目录。后续启动将直接重用现有数据。

### API 密钥（加密）

适用于程序化 API 访问。密钥以 GPG（AES-256）加密存储，不会以明文形式保存。

```bash
# 1. Create plaintext .env
cat <<EOF > .env.keys
OPENAI_API_KEY=sk-xxxxx
EOF

# 2. Encrypt (you will be prompted to set a passphrase)
encrypt_env.sh    # available inside container, or ./encrypt_env.sh on host

# 3. Remove plaintext
rm .env.keys
```

容器启动时，若检测到工作区中存在 `.env.gpg`，系统将提示输入密码。解密后的密钥仅以环境变量的形式保存于内存中。

> **注意：** `.env` 与 `.env.gpg` 已加入 `.gitignore`。

## 作为 Subtree 使用

此 repo 可通过 `git subtree` 嵌入至其他项目，让项目自带 Docker 开发环境。

### 添加到你的项目

```bash
git subtree add --prefix=docker/codex_cli \
    https://github.com/ycpss91255-docker/codex_cli.git main --squash
```

添加后的目录结构示例：

```text
my_project/
├── src/                         # 项目源代码
├── docker/codex_cli/            # Subtree
│   ├── build.sh
│   ├── run.sh
│   ├── compose.yaml
│   ├── Dockerfile
│   └── .base/
└── ...
```

### 构建与运行

```bash
cd docker/codex_cli
./build.sh && ./run.sh
```

`build.sh` 内部使用 `--base-path`，因此无论从何处执行，路径检测都能正确工作。

### 工作区检测行为

<details>
<summary>展开查看作为 subtree 使用时的检测行为</summary>

当 subtree 位于 `my_project/docker/codex_cli/` 时：

- **IMAGE_NAME**：目录名称为 `codex_cli`（非 `docker_*`），因此检测会回退至 `.env.example`，其中设置了 `IMAGE_NAME=codex_cli` — 可正常工作。
- **WS_PATH**：策略 1（同层扫描）与策略 2（向上遍历）可能无法匹配，因此策略 3（回退）会解析至上层目录（`my_project/docker/`）。

**建议**：首次构建后，请编辑 `.env` 中的 `WS_PATH` 指向实际工作区。此值在后续构建中会被保留。

</details>

### 同步上游更新

```bash
git subtree pull --prefix=docker/codex_cli \
    https://github.com/ycpss91255-docker/codex_cli.git main --squash
```

> **注意事项**：
> - 本地修改会由 git 正常跟踪。
> - 若上游修改了与你本地相同的文件，`subtree pull` 可能会产生合并冲突。
> - **不要**修改 subtree 内的 `.base/` — 它由 env repo 自身的 subtree 管理。

## 设置

每次执行 `build.sh` / `run.sh` 时会自动生成 `.env`（可传入 `--no-env` 跳过）。详情请参阅 [.env.example](.env.example)。

| 变量 | 说明 |
|------|------|
| `USER_NAME` / `USER_UID` / `USER_GID` | 与主机相符的容器用户（自动检测） |
| `GPU_ENABLED` | 自动检测，用于设置 `BASE_IMAGE` 与 `GPU_VARIANT` |
| `BASE_IMAGE` | `node:20-slim`（CPU）或 `nvidia/cuda:13.1.1-cudnn-devel-ubuntu24.04`（GPU） |
| `WS_PATH` | 挂载至容器内 `~/work` 的主机路径 |
| `IMAGE_NAME` | Docker 镜像名称（默认：`codex_cli`） |

## Smoke Tests

详见 [TEST.md](../test/TEST.md)。

## 架构

```
.
├── Dockerfile             # Multi-stage build (sys -> base -> devel -> test)
├── compose.yaml           # Services: devel (CPU), devel-gpu, test
├── build.sh               # Build with auto .env generation
├── run.sh                 # Run with auto .env generation
├── exec.sh                # Exec into running container
├── entrypoint.sh          # DinD startup, OAuth copy, API key decryption
├── encrypt_env.sh         # Helper to encrypt API keys
├── post_setup.sh          # Derives BASE_IMAGE from GPU_ENABLED
├── .env.example           # Template for .env
├── smoke/            # Bats smoke tests
│   ├── codex_env.bats
│   └── test_helper.bash
├── .base/   # Auto .env generator (git subtree)
├── README.md
└── README.zh-TW.md
```

### Dockerfile 阶段

| 阶段 | 用途 |
|------|------|
| `sys` | 用户/群组创建、语言环境、时区、Node.js（仅 GPU 版本） |
| `base` | 开发工具、Python、构建工具、Docker、jq、ripgrep |
| `devel` | Codex CLI、入口点、非 root 用户 |
| `test` | Bats smoke test（临时性，验证后舍弃） |

### Compose 服务

| 服务 | 说明 |
|------|------|
| `devel` | CPU 版本（默认） |
| `devel-gpu` | 附 NVIDIA 设备保留的 GPU 版本 |
| `test` | smoke test（以 profile 控制） |

### 入口点流程

1. 通过 sudo 启动 `dockerd`（DinD），等待就绪（最多 30 秒）
2. 将只读挂载的 OAuth 凭证复制至 `data/` 目录（仅首次执行）
3. 解密 `.env.gpg` 并将 API 密钥导出为环境变量（若存在）
4. 执行 CMD（`bash`）

### 预安装工具

| 工具 | 用途 |
|------|------|
| Codex CLI | OpenAI AI CLI |
| Docker (DinD) | 容器内的隔离 Docker daemon |
| Node.js 20 | CLI 工具的执行环境 |
| Python 3 | 脚本开发 |
| git, curl, wget | 版本控制与下载 |
| jq, ripgrep | JSON 处理与代码搜索 |
| make, g++, cmake | 构建工具链 |
| tree | 目录可视化 |

GPU 版本另外包含：CUDA 13.1.1、cuDNN、OpenCL、Vulkan。

### 容器能力

两种服务皆需要 `SYS_ADMIN`、`NET_ADMIN`、`MKNOD` 能力，并搭配 `seccomp:unconfined`，以确保 DinD 正常运作。内部 Docker daemon 与主机完全隔离。
