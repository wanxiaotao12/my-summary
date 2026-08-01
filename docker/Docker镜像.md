# Docker 镜像

## 什么是镜像

镜像（Image）是一个**只读的文件系统快照**，包含了运行应用所需的代码、运行时、库、环境变量和配置文件。容器（Container）是镜像的运行实例。

类比理解：镜像 ≈ 类（Class），容器 ≈ 对象（Object）。

## 镜像名的组成

完整镜像引用格式：

```
[Registry地址[:端口]/]Repository[:Tag]
```

各部分说明：

| 部分 | 说明 | 是否必填 | 默认值 | 示例 |
| --- | --- | --- | --- | --- |
| Registry 地址 | 镜像仓库服务器的地址，决定从哪里拉取镜像 | 否 | `docker.io`（Docker Hub） | `registry.cn-hangzhou.aliyuncs.com` |
| Repository | 镜像仓库名（即镜像名），标识 Registry 中的具体镜像 | 是 | — | `library/nginx`、`xprobe/xinference` |
| Tag | 版本标签，区分同一镜像的不同版本 | 否 | `latest` | `8.0.39-debian`、`v0.12.3` |

### Repository 的结构

Repository 是两段式名称：`<命名空间>/<软件名>`

| 场景 | 格式 | 示例 |
| --- | --- | --- |
| Docker Hub 官方镜像 | 省略命名空间，默认 `library` | `nginx` → `library/nginx` |
| Docker Hub 用户镜像 | `<用户名>/<软件名>` | `xprobe/xinference` |
| 私有 Registry | `<Registry地址>/<命名空间>/<软件名>` | `registry.cn-hangzhou.aliyuncs.com/myteam/myapp` |

### 完整示例解析

```
registry.cn-hangzhou.aliyuncs.com:5000/myteam/myapp:v1.2
├── Registry ─────────────────────────┤│       │     │
│   地址: registry.cn-hangzhou.aliyuncs.com     │     │
│   端口: 5000                                  │     │
├── Repository ─────────────────────────────────┤     │
│   命名空间: myteam                            │     │
│   软件名: myapp                               │     │
└── Tag ──────────────────────────────────────────────┘
    版本: v1.2
```

最简形式：

```
nginx
├── Registry: docker.io（默认）
├── Repository: library/nginx（默认 library）
└── Tag: latest（默认）
```

## 镜像的分层存储

镜像由多个**只读层（Layer）** 叠加而成，每层对应 Dockerfile 中的一条指令（如 `RUN`、`COPY`）。

```
┌─────────────────────┐  ← 容器层（可写，容器运行时产生）
├─────────────────────┤
│  COPY app.jar       │  ← 镜像层 3
├─────────────────────┤
│  RUN apt-get install│  ← 镜像层 2
├─────────────────────┤
│  FROM ubuntu:22.04  │  ← 镜像层 1（基础镜像）
└─────────────────────┘
```

- **层共享**：多个镜像可共享相同的基础层，节省磁盘空间
- **拉取增量**：`docker pull` 时只下载本地缺失的层
- **不可变**：镜像层一旦创建不可修改，修改会产生新层

## 镜像的标识

| 标识方式 | 格式 | 示例 |
| --- | --- | --- |
| 名称 + 标签 | `Repository:Tag` | `nginx:1.25` |
| Image ID | sha256 前 12 位 | `a8758716bb6a` |
| Digest | `Repository@sha256:完整哈希` | `nginx@sha256:4c0fdaa8b634...` |

> **Tag 可变，Digest 不可变**。同一个 Tag（如 `latest`）可能指向不同的镜像内容；Digest 则精确锁定某一次构建产物，适合生产环境固定版本。
