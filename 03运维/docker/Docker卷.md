# Docker 卷（Volume）

## 什么是 Docker 卷

Docker 卷（Volume）是 Docker 提供的数据持久化机制，用于在宿主机和容器之间共享、持久化数据。

容器文件系统是临时的：容器删除后，容器内产生的所有数据随之丢失。Docker 卷将数据存储在容器文件系统之外（宿主机上），即使容器被删除，数据依然保留。

## 作用

| 场景 | 说明 |
|------|------|
| 数据持久化 | 数据库、日志等数据在容器删除后不丢失 |
| 数据共享 | 多个容器同时挂载同一个卷，实现数据共享 |
| 开发调试 | 将宿主机代码目录挂载到容器，修改代码后容器内即时生效 |
| 配置管理 | 将宿主机的配置文件挂载到容器，避免每次修改都要重建镜像 |

## 为什么需要卷

容器生命周期和数据生命周期是分离的：

- **没有卷**：数据存储在容器可写层（container filesystem），容器删除 = 数据删除
- **有卷**：数据存储在宿主机上，容器删除 ≠ 数据删除

```
没有卷时：
  容器运行 → 产生数据 → 容器删除 → 数据丢失

有卷时：
  容器运行 → 产生数据 → 数据写入卷（宿主机） → 容器删除 → 数据仍在卷中
```

## 常用命令

### 卷管理

#### 列出所有卷

```bash
docker volume ls
```

输出示例：

```
DRIVER    VOLUME NAME
local     my_data
local     a8f3b2c1d4e5f6...
```

第一行 `DRIVER    VOLUME NAME` 是表头，之后每行是一个卷的信息：

| 字段 | 说明 |
|------|------|
| DRIVER | 存储驱动，`local` 表示存储在本地磁盘 |
| VOLUME NAME | 卷名；有名字的是命名卷，纯 ID 的是匿名卷 |

#### 创建命名卷

```bash
docker volume create <卷名>
```

执行后 Docker 在宿主机上自动创建对应目录（Linux 下为 `/var/lib/docker/volumes/<卷名>/_data`）。如果不提前创建，在 `docker run -v` 中使用一个不存在的卷名时，Docker 也会自动创建该卷。

#### 查看卷详情

```bash
docker volume inspect <卷名>
```

输出示例：

```json
[
    {
        "CreatedAt": "2026-08-04T18:00:00+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/my_data/_data",
        "Name": "my_data",
        "Options": null,
        "Scope": "local"
    }
]
```

关键字段：

| 字段 | 说明 |
|------|------|
| Name | 卷的名称 |
| Mountpoint | 宿主机上数据实际存储的路径 |
| Driver | 存储驱动，`local` 表示存在本地磁盘 |

#### 删除卷

```bash
docker volume rm <卷名>
```

删除卷的同时，**宿主机上对应的数据文件也会被删除**（`/var/lib/docker/volumes/<卷名>/_data/` 下的所有内容），数据无法恢复。

如果有容器正在使用该卷，删除会失败：

```bash
docker volume rm my_data
# Error: remove my_data: volume is in use - delete underlying containers first
```

需要先停止并删除使用该卷的容器，再删除卷。

> **注意**：如需保留数据，请先备份再删除。

**命名卷和绑定挂载的删除行为对比**：

| 操作 | 命名卷 | 绑定挂载 |
|------|--------|----------|
| 删除容器 | 卷不受影响，数据保留 | 宿主机文件不受影响 |
| 删除卷（`docker volume rm`） | 宿主机数据被删除 | 不适用（绑定挂载没有卷） |
| 手动删除宿主机文件 | — | 容器内挂载点的数据同步消失 |

#### 删除未使用的卷

```bash
docker volume prune
```

删除所有未被任何容器引用的卷（包括命名卷和匿名卷）。执行后会提示确认，加 `-f` 可跳过确认。

## 挂载方式

Docker 提供三种挂载方式：

| 方式 | 说明 | 适用场景 |
|------|------|----------|
| 命名卷（Named Volume） | 有名字的卷，Docker 自动管理宿主机存储路径 | 数据持久化，推荐用于生产环境 |
| 匿名卷（Anonymous Volume） | 没有名字的卷，Docker 自动生成 ID 作为标识 | 临时场景，不推荐 |
| 绑定挂载（Bind Mount） | 将宿主机指定路径挂载到容器 | 开发调试、配置文件挂载 |

### 命名卷（Named Volume，有名字的卷）

创建命名卷时，你只指定卷名，**数据在宿主机上的存储路径由 Docker 自动分配和管理**。Docker 会在宿主机上创建对应的目录，将容器内的数据写入该目录。你不需要指定宿主机路径，也不能自定义路径，只能通过 `docker volume inspect` 查看 Docker 分配的路径（Linux 下为 `/var/lib/docker/volumes/<卷名>/_data`）。

```bash
# 创建命名卷
docker volume create my_data

# 查看卷详情（含宿主机实际路径）
docker volume inspect my_data
# 输出示例：
# "Mountpoint": "/var/lib/docker/volumes/my_data/_data"
```

使用命名卷启动容器时，`-v` 后面只写**卷名:容器路径**，不指定宿主机路径（由 Docker 自动管理）：

```bash
# 使用命名卷启动容器
docker run -d --name mysql -v my_data:/var/lib/mysql mysql:8.0
#                       ^^^^^^^ 卷名（Docker 自动管理宿主机路径）
#                              ^^^^^^^^^^^^^^ 容器内路径
```

**如何区分命名卷和绑定挂载**：看 `-v` 左边是否以 `/` 开头：

```bash
# 不以 / 开头 → 命名卷（Docker 自动管理宿主机路径）
docker run -v my_data:/var/lib/mysql mysql:8.0

# 以 / 开头 → 绑定挂载（宿主机路径由你指定）
docker run -v /home/user/data:/var/lib/mysql mysql:8.0
```

**优点**：
- Docker 统一管理，跨平台路径一致
- 可通过 `docker volume` 命令备份、迁移
- 容器删除后数据自动保留

### 匿名卷（Anonymous Volume，没有名字的卷）

不指定宿主机路径和卷名，Docker 自动生成唯一 ID 作为卷名。**不需要提前创建**，在 `docker run` 时直接 `-v /data`（只写容器内路径，不写冒号左边），Docker 会自动创建一个匿名卷并挂载。

```bash
# 使用匿名卷（只写容器内路径，Docker 自动创建匿名卷）
docker run -d --name app -v /data nginx:latest
#                        ^^^^^^^ 只有容器路径，没有卷名和宿主机路径

# 查看时显示为 ID 而非名称
docker volume ls
# DRIVER    VOLUME NAME
# local     a8f3b2c1d4e5f6...
```

**缺点**：难以追踪和管理（没有名字，只有 ID），删除困难，不推荐使用。

### 绑定挂载

将宿主机的任意路径直接挂载到容器内指定路径。

**优点**：
- 可直接指定宿主机路径，便于定位和管理
- 适合开发时挂载源代码目录

**缺点**：
- 路径与宿主机强耦合，跨平台不兼容（Windows/macOS/Linux 路径不同）
- 宿主机路径不存在时，Docker 会自动创建目录（而非文件），可能导致意外

## 命名卷 vs 绑定挂载

| 对比项 | 命名卷 | 绑定挂载 |
|--------|--------|----------|
| 存储路径 | Docker 自动管理 | 用户指定宿主机路径 |
| 跨平台兼容 | 是 | 否（路径格式不同） |
| 容器删除后数据保留 | 是 | 是（数据在宿主机） |
| 适用场景 | 生产环境数据持久化 | 开发调试、配置文件挂载 |
| 管理方式 | `docker volume` 命令 | 直接操作宿主机文件系统 |
| 自动创建 | 容器启动时自动创建 | 路径不存在时自动创建（可能创建目录而非文件） |

**选择建议**：

| 场景 | 推荐方式 | 原因 |
|------|----------|------|
| 开发时挂载源代码 | 绑定挂载 | 修改代码后容器内即时生效，方便调试 |
| 挂载配置文件 | 绑定挂载 | 直接编辑宿主机上的配置文件即可 |
| 数据库、缓存等数据持久化 | 命名卷 | Docker 统一管理，不依赖宿主机路径 |
| 容器编排（K8s/Swarm） | 命名卷 | 与编排工具集成，自动管理生命周期 |

简单记：**开发用绑定挂载，生产用命名卷**。

## 挂载实践

### 在 docker run 中挂载命名卷

**格式**：`-v <卷名>:<容器路径>`

| 参数 | 说明 |
|------|------|
| 卷名 | 命名卷的名称，不以 `/` 开头 |
| 容器路径 | 容器内的挂载路径 |

```bash
# 挂载命名卷
docker run -d --name db -v db_data:/var/lib/postgresql/data postgres:15
#                       ^^^^^^^ 卷名（不以 / 开头）
#                              ^^^^^^^^^^^^^^^^^^^^^^ 容器内路径
```

### 在 docker run 中绑定挂载

**格式**：`-v <宿主机路径>:<容器路径>[:权限]`

| 参数 | 说明 |
|------|------|
| 宿主机路径 | 宿主机上的路径，以 `/` 开头 |
| 容器路径 | 容器内的挂载路径 |
| 权限 | 可选，不填默认为 `rw`：<br>`rw` — 读写（默认）<br>`ro` — 只读 |

```bash
# 挂载目录
docker run -d --name app -v /home/user/data:/app/data nginx:latest

# 挂载单个文件
docker run -d --name app -v /home/user/nginx.conf:/etc/nginx/nginx.conf nginx:latest

# 只读挂载（容器内只能读，不能写）
docker run -d --name app -v /home/user/config:/app/config:ro nginx:latest
```

### 在 Docker Compose 中挂载数据

在 `docker-compose.yml` 的 `services` 下通过 `volumes` 配置挂载，支持命名卷和绑定挂载两种方式。

**格式**：

```yaml
volumes:
  - <宿主机路径或卷名>:<容器路径>[:权限]
```

| 参数 | 是否可选 | 说明 |
|------|----------|------|
| 宿主机路径或卷名 | 必填 | 通过开头字符判断类型：<br>以 `./` 或 `/` 开头 → 绑定挂载（宿主机路径）<br>其他 → 命名卷（卷名） |
| 容器路径 | 必填 | 容器内的挂载路径 |
| 权限 | 可选 | 不填默认为 `rw`：<br>`rw` — 读写（默认）<br>`ro` — 只读 |

**宿主机路径的说明**：

| 路径类型 | 说明 | 示例 |
|----------|------|------|
| 相对路径 | 以 `./` 开头，相对宿主机上 `docker-compose.yml` 所在目录 | `./config` |
| 绝对路径 | 以 `/` 开头，宿主机上的完整路径 | `/home/user/config` |

```yaml
services:
  db:
    image: postgres:15
    volumes:
      - db_data:/var/lib/postgresql/data    # 命名卷
      - ./config:/app/config:ro             # 绑定挂载（只读）

volumes:
  db_data:                                  # 声明命名卷
```

`volumes` 声明写在文件底部，但 `services` 中在上面就可以直接引用。Docker Compose 会先解析整个文件的结构，再创建资源，所以不存在先后顺序的问题。

`volumes` 声明不是必须的。不声明时，Docker Compose 会自动创建卷，但会提示警告（orphan volume）。声明的作用：

1. **消除警告** — 告诉 Docker Compose 该卷属于当前项目，不是孤立卷
2. **统一管理** — 执行 `docker compose down --volumes` 时可一起清理项目相关卷

## 数据迁移与备份

本章节介绍**命名卷**的备份和迁移方法。绑定挂载的数据直接存储在宿主机的指定路径上，无需特殊迁移，直接用 `cp`、`tar`、`scp` 等系统命令操作即可。

迁移命名卷时，不需要知道 Docker 把数据存在宿主机的哪个路径。方法是：启动一个临时容器，同时挂载源卷和目标路径，用 `tar` 完成打包。

### 备份命名卷

```bash
# 创建一个临时容器，同时挂载源卷和宿主机备份目录
docker run --rm \
  -v my_data:/source:ro \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/my_data_backup.tar.gz -C /source .
```

原理说明：

| 参数 | 作用 |
|------|------|
| `-v my_data:/source:ro` | 把要备份的卷挂载到临时容器的 `/source`（只读） |
| `-v $(pwd):/backup` | 把宿主机当前目录挂载到临时容器的 `/backup`（存放备份文件） |
| `tar czf ...` | 在容器内把 `/source` 打包到 `/backup/my_data_backup.tar.gz` |

执行后，`my_data_backup.tar.gz` 会出现在你执行命令的宿主机目录下。

### 恢复命名卷

```bash
# 将备份恢复到新的卷
docker run --rm \
  -v my_data_new:/target \
  -v $(pwd):/backup \
  alpine \
  sh -c "cd /target && tar xzf /backup/my_data_backup.tar.gz"
```

### 跨机器迁移

```bash
# 第 1 步：在源机器上备份
docker run --rm -v my_data:/source:ro -v $(pwd):/backup alpine tar czf /backup/my_data_backup.tar.gz -C /source .

# 第 2 步：把 tar.gz 文件拷贝到目标机器（scp / rsync 等）
scp my_data_backup.tar.gz user@remote:/tmp/

# 第 3 步：在目标机器上恢复
docker run --rm -v my_data_new:/target -v /tmp:/backup alpine sh -c "cd /target && tar xzf /backup/my_data_backup.tar.gz"

# 第 4 步：启动容器使用新卷
docker run -d --name mysql -v my_data_new:/var/lib/mysql mysql:8.0
```

### 直接访问宿主机路径（不推荐）

你也可以通过 `inspect` 查到宿主机路径后直接操作，但这种方式依赖 Docker 内部路径，不跨平台：

```bash
# 查看宿主机实际路径
docker volume inspect my_data
# "Mountpoint": "/var/lib/docker/volumes/my_data/_data"

# 直接拷贝（仅 Linux，不推荐用于迁移）
tar czf my_data_backup.tar.gz -C /var/lib/docker/volumes/my_data/_data .
```

## 注意事项

1. **macOS / Windows 用户注意**：Docker 运行在虚拟机中，绑定挂载的性能低于 Linux 原生。开发时只挂载必要的目录，避免挂载大目录。
2. **绑定挂载路径不存在时的行为**：如果宿主机路径不存在，Docker 会自动以 root 权限创建**目录**（即使你期望挂载的是文件）。挂载单个文件前，请确保文件已存在。
3. **命名卷不会随容器自动删除**：删除容器后，命名卷仍然保留。如需清理，使用 `docker volume prune`。
4. **卷的权限问题**：绑定挂载时，容器内进程的 UID 可能与宿主机文件所有者不一致，导致权限错误。可通过 `:ro` 限制写入，或在 Dockerfile 中调整用户。
