# Dockerfile 常用指令指南

## 什么是 Dockerfile

Dockerfile 是一个纯文本文件，通过一系列指令定义如何构建 Docker 镜像。它是镜像的"配方"，任何人使用同一个 Dockerfile 都能构建出完全一致的镜像。

### 为什么需要 Dockerfile

Dockerfile 的核心目的是**构建镜像**——把"如何把应用打包成镜像"这件事用文件固化下来。在此基础上，它还带来三个好处：

- **可重复性**：消除"在我机器上能跑"的问题，任何人用同一个 Dockerfile 构建出的镜像完全一致
- **版本控制**：用 Git 管理镜像配置，追踪变更历史
- **自动化**：CI/CD 流水线中自动构建，无需人工干预

### Dockerfile 在工作流中的位置

从编写 Dockerfile 到应用跑起来，分为三步，Dockerfile 处于第一步，是后两步的输入：

```
第一步              第二步            第三步
编写 Dockerfile  ──▶  构建镜像  ──▶  运行容器
（构建配方）         （可分发的产物）   （运行中的实例）
```

- **第一步：编写 Dockerfile**——写好镜像的"构建配方"，定义基础镜像、要复制的文件、要执行的命令等。
- **第二步：构建镜像**——使用 `docker build` 命令执行 Dockerfile，Docker 会逐条读取并执行其中的指令（FROM、RUN、COPY 等），最终生成一个可分发、可版本化的镜像。
- **第三步：运行容器**——使用 `docker run` 命令基于镜像启动一个容器，应用真正在容器中跑起来。

### 文件特征

| 属性 | 说明 |
|------|------|
| 文件名 | 默认为 `Dockerfile`（无扩展名，首字母大写 D）；也可用其他名字，构建时用 `-f` 指定 |
| 文件格式 | 纯文本，每行一条指令 |
| 编码 | UTF-8 |

### Dockerfile 存放位置

Dockerfile 是一个普通文本文件，可以存放在**任意目录**。以下是常见场景：

#### 场景一：有项目时

放在项目根目录，文件名固定为 `Dockerfile`（无扩展名）。

```
my-project/
├── Dockerfile              # Dockerfile 文件
├── .dockerignore           # 排除文件配置
├── app.py                  # 应用代码
├── requirements.txt        # 依赖文件
└── README.md
```

#### 场景二：没有项目时

可以单独创建一个目录存放 Dockerfile，用于学习或测试。

```
docker-learning/
├── Dockerfile              # Dockerfile 文件
└── app.py                  # 测试用的应用代码
```

**为什么推荐放在项目根目录**：
- `docker build` 默认在当前目录查找名为 `Dockerfile` 的文件
- COPY 指令复制的文件路径都是相对于 Dockerfile 所在目录

**也可以自定义文件名和位置**：

```bash
# 指定其他文件名
docker build -f Dockerfile.prod .

# 指定其他路径
docker build -f docker/Dockerfile .
```

## 常用指令

Dockerfile 由一系列指令组成，每条指令定义镜像构建的一个步骤。以下是 Dockerfile 中最常用的指令，按在 Dockerfile 中的典型使用顺序排列。

| 指令 | 一句话作用 |
|------|-----------|
| `FROM` | 指定基于哪个基础镜像构建 |
| `WORKDIR` | 设置后续指令的工作目录 |
| `RUN` | 构建镜像时执行 Linux 命令 |
| `COPY` | 将本地文件复制到镜像 |
| `ENV` | 设置环境变量 |
| `EXPOSE` | 声明容器监听的端口 |
| `CMD` | 容器启动时的默认命令 |
| `ENTRYPOINT` | 容器启动时的固定入口命令 |
| `ARG` | 定义构建时变量 |
| `VOLUME` | 声明数据卷挂载点 |
| `USER` | 指定容器运行的用户 |

**注意**：Dockerfile 中几乎每条指令都会生成一层镜像层（RUN、COPY、ADD、ENV、WORKDIR 等），镜像层数越多，镜像体积越大。

### FROM — 指定基础镜像

每个 Dockerfile 必须以 FROM 开头，指定镜像基于哪个已有镜像构建。

#### 命令格式

```
FROM [仓库名/]镜像名[:标签]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 仓库名（可选） | 镜像所在的仓库，不填默认从 Docker Hub 拉取 | `library/`、`node` |
| 镜像名 | 镜像的名称 | `python`、`node`、`openjdk` |
| 标签（可选） | 镜像的版本标签，不填默认使用 `latest` | `3.9-slim`、`18-alpine` |

#### 命令示例

```dockerfile
FROM python:3.9-slim
```

#### 命令示例解析

| 完整写法 | 仓库名 | 镜像名 | 标签 |
|--------|--------|--------|------|
| `python:3.9-slim` | Docker Hub 官方 | python | 3.9-slim |
| `node:18-alpine` | Docker Hub 官方 | node | 18-alpine |
| `openjdk:17-slim` | Docker Hub 官方 | openjdk | 17-slim |
| `myregistry.com/myapp:v1.0` | myregistry.com | myapp | v1.0 |

**常用基础镜像**：

| 基础镜像 | 适用场景 |
|----------|----------|
| `python:3.9-slim` | Python 应用，体积较小 |
| `node:18-alpine` | Node.js 应用，体积极小 |
| `openjdk:17-slim` | Java 应用 |
| `golang:1.21` | Go 编译环境 |
| `scratch` | 空镜像，用于编译后的静态二进制 |

### WORKDIR — 设置工作目录

设置后续 RUN、CMD、COPY 指令的工作目录。如果设置的目录不存在，Docker 会自动创建该目录。

**关于 WORKDIR 的说明**：
- **位置**：WORKDIR 可以在 Dockerfile 中任意位置使用，但通常放在 FROM 之后、RUN/COPY 之前，因为它会影响后续指令的行为
- **默认值**：如果没有设置 WORKDIR，默认工作目录是镜像的根目录 `/`
- **作用范围**：WORKDIR 设置的工作目录既影响**镜像构建时**（RUN、COPY 等指令的执行目录），也影响**容器运行时**（通过 `docker exec` 进入容器时的默认目录）

#### 命令格式

```
WORKDIR [目录路径]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 目录路径 | 镜像中的目录路径，支持绝对路径 | `/app`、`/home/user/project` |

#### 命令示例

```dockerfile
WORKDIR /app
```

#### 命令示例解析

| 命令 | 含义 |
|------|------|
| `WORKDIR /app` | 将工作目录设置为 /app，后续指令都在 /app 下执行 |

**WORKDIR 的影响**：设置 WORKDIR 后，COPY 和 RUN 中的 `.` 都代表 WORKDIR 设置的目录（如下例中的 `/app`）。

```dockerfile
WORKDIR /app
COPY . .
RUN pip install .
```

以上两条指令等价于：

```dockerfile
COPY . /app
RUN cd /app && pip install .
```

### RUN — 执行命令

在构建镜像时执行 Linux 命令。

#### 命令格式

```
RUN [命令]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 命令 | 任意 Linux shell 命令 | `pip install -r requirements.txt`、`apt-get update` |

#### 命令示例

```dockerfile
RUN pip install -r requirements.txt
RUN apt-get update && apt-get install -y curl
```

#### 命令示例解析

| 命令 | 含义 |
|------|------|
| `RUN pip install -r requirements.txt` | 在镜像中安装 Python 依赖 |
| `RUN apt-get update && apt-get install -y curl` | 更新包源并安装 curl |

**最佳实践**：用 `&&` 合并多条 RUN 命令，减少镜像层数。

```dockerfile
# 推荐：合并为一层
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# 不推荐：生成三层
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

### COPY — 复制文件到镜像

将本地文件复制到镜像中。

#### 命令格式

```
COPY [--from=<阶段名>] [源路径] [目标路径]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| `--from=<阶段名>`（可选） | 指定复制来源为另一个构建阶段；不写则默认从本地电脑（构建上下文）复制 | `--from=build` |
| 源路径 | 文件路径<br>1. 默认从本地电脑复制：相对路径（相对于 Dockerfile 所在目录）或绝对路径<br>2. 带 `--from` 时：源文件在指定阶段里的路径 | `requirements.txt`（单个文件）<br>`src/`（整个目录，递归复制）<br>`.`（Dockerfile 所在目录下所有文件，受 .dockerignore 排除） |
| 目标路径 | 镜像中的目标路径 | `.`（由 WORKDIR 设置的工作目录，未设置时默认为 `/`）<br>`/app/`<br>`/app/config/` |

**目录复制示例**：假设项目结构如下：

```
my-project/
├── Dockerfile
├── src/
│   ├── main.py
│   └── utils/
│       └── helper.py
```

执行 `COPY src /app/src` 后，镜像中的结构为：

```
/app/src/
├── main.py
└── utils/
    └── helper.py
```

可以看到，`src/` 下的子目录 `utils/` 和其中的文件 `helper.py` 也会被一起复制。

#### 命令示例

```dockerfile
COPY requirements.txt .
COPY . .
```

#### 命令示例解析

| 命令 | 源路径 | 目标路径 | 含义 |
|------|--------|----------|------|
| `COPY requirements.txt .` | `requirements.txt` | `.`（当前工作目录） | 将 requirements.txt 复制到镜像的工作目录 |
| `COPY . .` | `.`（Dockerfile 所在目录下的所有文件） | `.`（当前工作目录） | 将 Dockerfile 所在目录下的所有文件复制到镜像的工作目录 |
| `COPY src /app/src` | `src/` 目录 | `/app/src` | 将 src 目录复制到镜像的 /app/src |

**`.` 的含义**：代表 WORKDIR 设置的当前工作目录。如果没有设置 WORKDIR，默认是镜像根目录 `/`。

**COPY 与 ADD 的区别**：

| 指令 | 区别 |
|------|------|
| `COPY` | 仅复制文件/目录 |
| `ADD` | 除复制外，支持 URL 下载和自动解压 tar |

**推荐优先使用 COPY**，行为更明确。

#### 从其他构建阶段复制（`--from`）

在多阶段构建中，COPY 可以用 `--from` 从另一个构建阶段复制文件，而不是从本地电脑复制：

```dockerfile
COPY --from=build /build/target/app.jar .
```

| 部分 | 含义 |
|------|------|
| `--from=build` | 复制来源是名为 `build` 的构建阶段（即 `FROM ... AS build` 那个阶段），而非本地电脑 |
| `/build/target/app.jar` | 源文件在 `build` 阶段里的路径 |
| `.` | 目标路径（当前阶段的当前目录） |

两种来源对比：

| 写法 | 复制来源 |
|------|----------|
| `COPY app.jar .` | 从本地电脑（构建上下文）复制 |
| `COPY --from=build /build/target/app.jar .` | 从另一个构建阶段复制 |

`--from` 是多阶段构建的核心，详见「多阶段构建」一节。

### ENV — 设置环境变量

在镜像中设置环境变量。这里设置的是**默认值**，运行容器时可用 `-e` 参数临时覆盖，无需重新构建镜像：

```bash
docker run -e APP_PORT=9000 my-app   # 覆盖为 9000，镜像本身不变
```

#### 命令格式

```
ENV [变量名]=[变量值]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 变量名 | 环境变量的名称 | `PYTHONUNBUFFERED`、`APP_PORT` |
| 变量值 | 环境变量的值 | `1`、`8000` |

#### 命令示例

```dockerfile
ENV PYTHONUNBUFFERED=1
ENV APP_PORT=8000
```

#### 命令示例解析

| 命令 | 含义 |
|------|------|
| `ENV PYTHONUNBUFFERED=1` | 设置 Python 输出不缓冲，日志实时打印 |
| `ENV APP_PORT=8000` | 设置应用端口为 8000 |

**在代码中使用**：设置后，应用代码中可以通过 `os.environ.get("APP_PORT")` 读取环境变量。

### EXPOSE — 声明端口

在镜像里写一条"备注"，声明容器打算使用哪个端口。**它本身不会让端口真正可访问**——真正开通端口靠的是运行时的 `-p` 映射（见下文）。

EXPOSE 的两个实际用处：

- **文档作用**：别人查看镜像时（`docker inspect`）能知道它设计用哪个端口，不用翻代码。
- **配合大写 `-P`**：运行时用 `docker run -P`，Docker 会自动把所有 EXPOSE 声明的端口随机映射到宿主机。

#### 命令格式

```
EXPOSE [端口号]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 端口号 | 容器内监听的端口 | `8000`、`8080`、`3000` |

#### 命令示例

```dockerfile
EXPOSE 8000
```

#### 命令示例解析

| 命令 | 含义 |
|------|------|
| `EXPOSE 8000` | 声明容器使用 8000 端口（只是备注，端口并未真正开通） |

**EXPOSE 与 -p 的区别**：

| 写法 | 作用 |
|------|------|
| `EXPOSE 8000`（Dockerfile 里） | 只是**声明/备注**，端口并不通 |
| `docker run -p 8000:8000`（运行时） | **真正**把端口映射出来，外部才能访问 |

**运行时端口映射**（这一步才让端口真正可访问）：

```bash
docker run -p 8000:8000 my-image
# 左边 8000：宿主机端口
# 右边 8000：容器端口
```

### CMD — 容器启动命令

容器启动时默认执行的命令。关键词是"默认"——如果运行时不指定命令就用它，指定了则让位（见下方注意事项）。

#### 命令格式

```
CMD ["命令", "参数1", "参数2"]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 命令 | 容器启动时执行的程序 | `python`、`java`、`node` |
| 参数 | 程序的参数 | `app.py`、`-jar app.jar`、`index.js` |

#### 命令示例

```dockerfile
CMD ["python", "app.py"]
```

#### 命令示例解析

| 命令 | 含义 |
|------|------|
| `CMD ["python", "app.py"]` | 容器启动时执行 `python app.py` |

#### 注意事项

CMD 存在两种"覆盖"，发生在不同时机，注意区分：

**1. 构建时：多条 CMD 只有最后一条生效**

Dockerfile 里写多条 CMD 时，前面的会被"顶掉"（不是累加，也不是报错），只有最后一条被保留：

```dockerfile
CMD ["python", "app.py"]
CMD ["python", "server.py"]
```

上面两条中，**只有最后一条 `CMD ["python", "server.py"]` 被保留**，容器启动时执行 `python server.py`；第一条在构建时即被丢弃，写了等于没写。

之所以这样设计，是因为一个容器只该有一个主进程（启动命令），所以多条 CMD 以最后一条为准。

**2. 运行时：`docker run` 可覆盖 CMD**

CMD 是"默认"命令，运行时若手动指定命令，CMD 就让位：

```bash
docker run my-image          # 没指定命令 → 用 CMD 里的命令
docker run my-image bash     # 指定了 bash → CMD 被顶掉，改跑 bash
```

### ENTRYPOINT — 容器入口点

与 CMD 类似，但不可被 `docker run` 的参数覆盖。CMD 的参数会作为 ENTRYPOINT 的附加参数。

#### 命令格式

```
ENTRYPOINT ["命令", "参数1", "参数2"]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 命令 | 容器启动时执行的程序 | `python`、`java`、`node` |
| 参数 | 程序的参数 | `app.py`、`-jar app.jar`、`index.js` |

#### 命令示例

```dockerfile
ENTRYPOINT ["python", "app.py"]
```

#### 命令示例解析

| 命令 | 含义 |
|------|------|
| `ENTRYPOINT ["python", "app.py"]` | 容器启动时固定执行 `python app.py`，不可被 docker run 覆盖 |

**CMD 与 ENTRYPOINT 的选择**：

| 场景 | 使用 |
|------|------|
| 固定命令，参数可变 | ENTRYPOINT + CMD（CMD 提供默认参数） |
| 默认命令，可被覆盖 | CMD |

### ARG — 定义构建时变量

定义只在**构建镜像时**（`docker build`）存在的变量，运行时不存在。常用来给构建过程传参，比如版本号。

#### 命令格式

```
ARG 变量名[=默认值]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 变量名 | 构建时变量的名称 | `VERSION`、`BUILD_ENV` |
| 默认值（可选） | 不传参时使用的值 | `1.0`、`prod` |

#### 命令示例

```dockerfile
ARG VERSION=1.0
RUN echo "building version ${VERSION}"
```

构建时通过 `--build-arg` 传入，覆盖默认值：

```bash
docker build --build-arg VERSION=2.0 -t my-app .
```

#### 命令示例解析

| 命令 | 含义 |
|------|------|
| `ARG VERSION=1.0` | 定义构建时变量 VERSION，默认值 1.0 |
| `--build-arg VERSION=2.0` | 构建时把 VERSION 覆盖为 2.0 |

**ARG 与 ENV 的区别**：

| 指令 | 生效时机 | 运行时是否存在 |
|------|----------|----------------|
| `ARG` | 仅构建时 | 否 |
| `ENV` | 构建时 + 运行时 | 是 |

常见组合：用 ARG 接收构建参数，再写进 ENV 让运行时也能用：

```dockerfile
ARG VERSION=1.0
ENV APP_VERSION=${VERSION}
```

### VOLUME — 声明数据卷挂载点

声明容器内的某个目录为数据卷，用于**数据持久化**——容器删除后，数据卷里的数据仍然保留。常用于数据库数据、日志等不能随容器消失的内容。

**注意：`VOLUME` 不是必须的**。它只是在镜像元数据里打个"建议标记"，告诉使用者这个目录适合做数据卷；真正决定数据存在哪里的是运行时的 `docker run -v`，且 `-v` **不依赖** `VOLUME`——即使 Dockerfile 里没写 `VOLUME`，照样可以用 `-v` 挂载。实际生产中，很多团队不写 `VOLUME`，直接在部署时用 `-v` 挂载。

#### 命令格式

```
VOLUME [目录路径]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 目录路径 | 容器内要挂载的目录 | `/data`、`/var/lib/mysql` |

#### 命令示例

```dockerfile
VOLUME /data
```

#### 命令示例解析

| 命令 | 含义 |
|------|------|
| `VOLUME /data` | 声明 /data 为数据卷，其中的数据在容器删除后仍保留 |

**运行时挂载到宿主机**：实际持久化数据通常用 `docker run -v` 把容器目录映射到宿主机目录，方便查看和备份（无需依赖上面的 `VOLUME` 声明）：

```bash
docker run -v /host/data:/data my-app
# 左边 /host/data：宿主机目录
# 右边 /data：容器内的数据卷目录
```

### USER — 指定运行用户

指定容器以哪个用户身份运行。**默认是 root**，出于安全考虑，生产环境通常切换到权限更低的普通用户，避免容器被攻破后获得 root 权限。

#### 命令格式

```
USER 用户名[:用户组]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 用户名 | 运行容器的用户 | `appuser`、`nginx` |
| 用户组（可选） | 用户所属的组 | `appuser:appgroup` |

#### 命令示例

```dockerfile
RUN useradd -m appuser
USER appuser
```

#### 命令示例解析

| 命令 | 含义 |
|------|------|
| `RUN useradd -m appuser` | 先创建一个名为 appuser 的用户 |
| `USER appuser` | 后续指令及容器运行时都以 appuser 身份执行 |

**注意**：USER 指定的用户必须已存在，所以通常先用 `RUN useradd` 创建，再用 `USER` 切换。

### 多阶段构建

一个 Dockerfile 里可以写**多个 FROM**，每个 FROM 开启一个独立的"构建阶段"，最终镜像只保留最后一个 FROM 那一层。

#### 命令格式

多阶段构建没有独立的指令，而是由 `FROM ... AS ...` 和 `COPY --from=...` 两个写法组合而成（以下为结构模板，尖括号内为占位符）：

```text
FROM <编译用基础镜像> AS <阶段名>
... 编译相关指令（COPY 源码、RUN 编译命令等）

FROM <运行用基础镜像>
... 运行相关指令
COPY --from=<阶段名> <编译产物路径> <目标路径>
```

| 写法 | 说明 |
|------|------|
| `FROM <镜像> AS <阶段名>` | 开启一个构建阶段，并用 `AS` 给它起别名（如 `AS build`），供后续阶段引用 |
| `COPY --from=<阶段名> ...` | 在后面的阶段里，从指定阶段复制文件（如编译产物） |

以下方的 Java 示例为例，它分为两个阶段：

| 阶段 | 基础镜像 | 作用 |
|------|----------|------|
| 第一阶段（`AS build`） | `maven:3.8-openjdk-17` | 编译：用完整的编译工具把源码编译成 `app.jar` |
| 第二阶段 | `openjdk:17-slim` | 运行：只放精简运行环境和编译产物 |

两个关键写法：

- **`AS build`**：给第一阶段起别名 `build`，供后面引用。
- **`COPY --from=build`**：从第一阶段把编译产物 `app.jar` 拷贝到第二阶段。

**为什么这么做**：编译需要 maven 等大量工具，但运行只需要一个 jar。多阶段构建把"编译的脏活"留在第一阶段，最终镜像不含编译工具，体积更小、更安全。完整的 Java 多阶段构建示例见 [Dockerfile使用指南.md](./Dockerfile使用指南.md)。

## 后续：怎么用起来

本文档只讲 Dockerfile **指令怎么写**。写好之后怎么构建镜像、怎么运行容器，以及完整示例、最佳实践等内容，见 [Dockerfile使用指南.md](./Dockerfile使用指南.md)。
