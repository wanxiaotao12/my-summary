# Docker 常用命令

## 基础

```bash
# 查看版本号
docker --version
```

```bash
# 查看 Docker 守护进程日志（仅 Linux systemd 环境）
journalctl -u docker -f
```

> macOS Docker Desktop 日志路径：`~/Library/Containers/com.docker.docker/Data/log/`

---

## 镜像

### 查看镜像

```bash
docker images
docker images | grep "gpt"
```

输出字段说明：

| 字段 | 含义 |
| --- | --- |
| Repository | 镜像名称，如 `nginx`、`mysql`、`xprobe/xinference` |
| Tag | 镜像版本标签（如 `latest`、`8.0`），与 Repository 组合构成完整引用 `nginx:latest` |
| Image ID | 镜像唯一标识（sha256 前 12 位） |
| Created | 镜像创建时间 |
| Size | 镜像大小 |

### 搜索镜像

Docker Hub：https://hub.docker.com/

### 拉取镜像

```bash
docker pull [选项] [Registry地址[:端口]/]仓库名[:标签]
```

- **Registry 地址**：不填默认为 Docker Hub（docker.io）
- **仓库名**：两段式 `<用户名>/<软件名>`，Docker Hub 下不填用户名默认为 `library`（官方镜像）

```bash
# 示例：拉取官方 ubuntu 18.04 镜像（等价于 library/ubuntu:18.04）
docker pull ubuntu:18.04

# 示例：指定标签
docker pull mysql:8.0.39-debian
```

> 下载过程按层（Layer）逐层拉取，结束后给出 sha256 摘要以校验一致性。

### 修改镜像 Tag

`docker tag` 不会修改镜像本身，只是新增一个标签引用（Image ID 相同）。

```bash
docker tag nginx:latest nginx:v1.0

# 删除旧标签（此时有 latest 和 v1.0 两个标签，删除 latest 后镜像仍保留）
docker rmi nginx:latest
```

> 如果镜像只有一个标签（如只有 `nginx:latest`），执行 `docker rmi nginx:latest` 会同时移除标签并删除镜像本体，释放磁盘空间。

### 删除镜像

`docker image rm` 与 `docker rmi` 完全等价。

```bash
# 按名称或 ID 删除
docker rmi <镜像名:标签 或 ImageID>

# 强制删除（镜像被容器引用时）
docker rmi -f <镜像名或ID>

# 删除多个
docker rmi image1 image2

# 删除所有镜像
docker rmi $(docker images -q)
```

#### 删除行为说明

该命令实际删除的是**标签（Tag）引用**，而非直接删除镜像本体：

| 场景 | 行为 |
| --- | --- |
| 镜像有多个 Tag | 只移除指定 Tag，镜像本体保留 |
| 镜像只剩最后一个 Tag | 移除 Tag 并删除镜像本体，释放磁盘空间 |
| 镜像被容器引用（含已停止） | 拒绝删除，需 `-f` 强制或先删容器 |

```bash
# 示例：nginx 有 latest 和 v1.0 两个标签
docker rmi nginx:latest
# 输出：Untagged: nginx:latest（镜像仍在，v1.0 可用）

docker rmi nginx:v1.0
# 输出：Untagged: nginx:v1.0 + Deleted: a8758716bb6a...（镜像真正删除）
```

> 强制删除（`-f`）不推荐用于有容器引用的场景，正确做法是先 `docker rm` 容器，再 `docker rmi` 镜像。

### 导入导出

```bash
# 导出（支持管道压缩）
docker save xprobe/xinference:v0.12.3 | gzip > inference_v0.12.3.tar.gz

# 导入
docker load -i inference_v0.12.3.tar.gz
```

---

## 容器

### 创建容器

#### 只创建不运行

```bash
docker create --name 容器名 镜像名:标签
```

#### 创建并运行（docker run）

```bash
# 后台守护进程方式
docker run -d --name 容器名 镜像名:标签

# 交互式方式（进入容器终端）
docker run -it --name 容器名 镜像名:标签 /bin/bash
```

> **注意**：通过 `run -it` 进入容器后，`exit` 会导致容器停止。再次进入需先 `docker start`，再用 `docker exec` 进入。

#### 常用参数

| 参数 | 说明 |
| --- | --- |
| `-d` / `--detach` | 后台运行 |
| `-i` | 保持 stdin 打开（交互模式） |
| `-t` | 分配伪终端（TTY），通常与 `-i` 搭配 |
| `--name` | 指定容器名称 |
| `-v` / `--volume` | 目录映射，格式：`宿主机路径:容器路径`，可多次使用 |
| `-p` / `--publish` | 端口映射，格式：`宿主机端口:容器端口`，可多次使用 |
| `-e` / `--env` | 设置环境变量，格式：`KEY=VALUE`，可多次使用 |

```bash
# 综合示例
docker run -d --name myapp -p 8080:80 -v /data:/app/data -e ENV=prod nginx:latest
```

### 查看容器

```bash
docker ps          # 正在运行的容器
docker ps -a       # 所有容器（含已停止）
docker ps -l       # 最近创建的容器
docker ps -a | grep "gpt"
```

### 启动 / 停止 / 重启

```bash
docker start <容器名或ID>
docker stop <容器名或ID>
docker restart <容器名或ID>

# 停止所有容器
docker stop $(docker ps -q)
```

### 删除容器

```bash
docker rm <容器名或ID>

# 删除所有容器
docker rm $(docker ps -a -q)
```

### 进入容器

```bash
# 推荐：exec（exit 后容器继续运行）
docker exec -it <容器名或ID> /bin/bash

# attach（exit 后容器停止）
docker attach <容器名或ID>
```

| 方式 | exit 后容器状态 |
| --- | --- |
| `exec` | 继续运行 |
| `attach` | 停止 |

### 退出容器

| 方式 | 效果 |
| --- | --- |
| `exit` | 退出终端；`-d` 启动的容器继续运行，`-it` 启动的容器停止 |
| `Ctrl+P` 然后 `Ctrl+Q` | 退出终端，容器始终继续运行 |

### 复制文件

```bash
docker cp <源路径> <目标路径>
```

源和目标可以是宿主机路径或 `容器名:容器内路径`，方向不限。

```bash
# 宿主机 → 容器
docker cp /root/boot.war my-centos:/usr/local/

# 容器 → 宿主机
docker cp my-centos:/usr/local/boot.war /root/
```

### 查看容器配置

```bash
docker inspect <容器名或ID>
```

### 查看日志

```bash
docker logs <容器名或ID>

# 实时跟踪
docker logs -f <容器名或ID>
```

---


---

## 系统清理

```bash
# 清理所有停止的容器、未使用的网络、悬空镜像和构建缓存
docker system prune

# 加上 -a 同时清理未被任何容器使用的镜像
docker system prune -a

# 查看磁盘占用
docker system df
```
