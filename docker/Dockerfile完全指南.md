# Dockerfile 完全指南

## 什么是 Dockerfile

Dockerfile 是一个纯文本文件，通过一系列指令定义如何构建 Docker 镜像。它是镜像的"配方"，任何人使用同一个 Dockerfile 都能构建出完全一致的镜像。

**文件特征**：

| 属性 | 说明 |
|------|------|
| 文件名 | 固定为 `Dockerfile`（无扩展名，首字母大写 D） |
| 文件格式 | 纯文本，每行一条指令 |
| 编码 | UTF-8 |

**为什么需要 Dockerfile**：
- **可重复性**：消除"在我机器上能跑"的问题，任何人构建的镜像一致
- **版本控制**：用 Git 管理镜像配置，追踪变更历史
- **自动化**：CI/CD 流水线中自动构建，无需人工干预

## Dockerfile 存放位置

Dockerfile 是一个普通文本文件，可以存放在**任意目录**。以下是常见场景：

### 场景一：有项目时

放在项目根目录，文件名固定为 `Dockerfile`（无扩展名）。

```
my-project/
├── Dockerfile              # Dockerfile 文件
├── .dockerignore           # 排除文件配置
├── app.py                  # 应用代码
├── requirements.txt        # 依赖文件
└── README.md
```

### 场景二：没有项目时

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

## 核心指令

Dockerfile 由一系列指令组成，每条指令定义镜像构建的一个步骤。以下是 Dockerfile 中最常用的核心指令，按在 Dockerfile 中的典型使用顺序排列。

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

| 镜像名 | 仓库名 | 镜像名 | 标签 |
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

### RUN — 执行命令

在构建镜像时执行 Linux 命令。

#### 命令格式

```
RUN [命令]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 命令 | 任意 Linux shell 命令 | `pip install -r requirements.txt`、`apt-get update` |

**注意**：Dockerfile 中几乎每条指令都会生成一层镜像层（RUN、COPY、ADD、ENV、WORKDIR 等），镜像层数越多，镜像体积越大。

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
COPY [源路径] [目标路径]
```

| 组成部分 | 说明 | 示例 |
|----------|------|------|
| 源路径 | 本地文件路径，可以是**相对路径**（相对于 Dockerfile 所在目录）或**绝对路径**，可以是**单个文件**、**整个目录**或**Dockerfile 所在目录** | `requirements.txt`（相对路径，单个文件）、`/app/src/`（绝对路径，整个目录）、`.`（Dockerfile 所在目录下的所有文件） |
| 目标路径 | 镜像中的目标路径 | `.`（当前工作目录）、`/app/`、`/app/config/` |

**源路径说明**：

| 源路径 | 类型 | 含义 |
|--------|------|------|
| `requirements.txt` | 单个文件 | 只复制 requirements.txt 这一个文件 |
| `src/` | 目录 | 复制 src 目录下的**所有内容**，包括子目录和子目录中的文件（递归复制） |
| `.` | Dockerfile 所在目录 | 复制 Dockerfile 所在目录下的所有文件（受 .dockerignore 排除） |

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

### WORKDIR — 设置工作目录

设置后续 RUN、CMD、COPY 指令的工作目录。如果目录不存在，自动创建。

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

**WORKDIR 的影响**：设置 WORKDIR 后，COPY 和 RUN 中的 `.` 都代表这个目录。

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

### ENV — 设置环境变量

在镜像中设置环境变量，容器运行时可覆盖。

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

声明容器监听的端口。**仅用于文档说明**，不真正暴露端口。运行时需用 `-p` 映射。

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
| `EXPOSE 8000` | 声明容器监听 8000 端口（仅文档说明，需运行时用 `-p` 映射） |

**运行时端口映射**：

```bash
docker run -p 8000:8000 my-image
# 左边 8000：宿主机端口
# 右边 8000：容器端口
```

### CMD — 容器启动命令

容器启动时默认执行的命令。**Dockerfile 中只能有一条 CMD 生效**，后面的会覆盖前面的。

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

**注意**：CMD 可以被 `docker run` 后面的命令覆盖。

```bash
# 覆盖了 CMD，容器启动时执行 bash
docker run my-image bash
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

## Dockerfile 使用方法

Dockerfile 的使用需要在**安装了 Docker 的环境**中执行。以下是常见的使用场景：

### 在哪里使用

| 环境 | 说明 |
|------|------|
| 本地电脑 | 在开发者的电脑上安装 Docker Desktop，用于本地开发和测试 |
| 服务器 | 在 Linux 服务器上安装 Docker，用于部署应用 |
| CI/CD 流水线 | 在 Jenkins、GitLab CI 等持续集成工具中自动构建镜像 |

**前提条件**：无论在哪种环境，都需要先安装 Docker。安装完成后，在终端中执行以下命令。

### 使用步骤

有了 Dockerfile 后，使用分三步：

**第一步：构建镜像**

在终端中，进入 Dockerfile 所在的目录，执行：

```bash
docker build -t my-app:v1.0 .
```

| 参数 | 说明 |
|------|------|
| `-t my-app:v1.0` | 给镜像命名并打标签 |
| `.` | 构建上下文路径，`.` 表示 Dockerfile 所在的目录 |

**第二步：查看镜像**

```bash
docker images
```

**第三步：运行容器**

```bash
docker run -p 8000:8000 my-app:v1.0
```

| 参数 | 说明 |
|------|------|
| `-p 8000:8000` | 端口映射，左边是宿主机端口，右边是容器端口 |
| `-d` | 后台运行 |
| `--name my-container` | 给容器命名 |

**完整运行示例**：

```bash
# 后台运行并命名
docker run -d --name my-app -p 8000:8000 my-app:v1.0

# 查看运行中的容器
docker ps

# 查看容器日志
docker logs my-app

# 停止容器
docker stop my-app
```

## 完整示例

以下是三种常见语言应用的 Dockerfile 文件完整示例，展示如何将上述指令组合写进一个 Dockerfile 文件中。

### Python 应用

以下是一个 Python 应用的 Dockerfile 文件内容：

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

### Java 应用

以下是一个 Java 应用的 Dockerfile 文件内容，使用多阶段构建：

```dockerfile
FROM maven:3.8-openjdk-17 AS build
WORKDIR /build
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM openjdk:17-slim
WORKDIR /app
COPY --from=build /build/target/app.jar .

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

### Node.js 应用

以下是一个 Node.js 应用的 Dockerfile 文件内容：

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["node", "index.js"]
```

## 最佳实践

### 1. 使用多阶段构建减小镜像体积

```dockerfile
FROM golang:1.21 AS build
WORKDIR /build
COPY . .
RUN go build -o myapp .

FROM alpine:latest
WORKDIR /app
COPY --from=build /build/myapp .
CMD ["./myapp"]
```

最终镜像只包含编译后的二进制，不包含编译工具链。

### 2. 使用 .dockerignore

```
.git
.gitignore
README.md
.env
*.log
```

排除不需要复制到镜像的文件，加快构建速度。

### 3. 利用构建缓存

```dockerfile
# 先复制依赖文件，再复制源代码
# 这样只有源代码变更时才会重新安装依赖
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### 4. 减小镜像体积

```dockerfile
# 安装后清理缓存
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# pip 不使用缓存
RUN pip install --no-cache-dir -r requirements.txt

# npm 只安装生产依赖
RUN npm ci --only=production
```

## 常用命令

| 命令 | 说明 |
|------|------|
| `docker build -t my-image:tag .` | 构建镜像 |
| `docker build -f Dockerfile.prod .` | 指定 Dockerfile 文件 |
| `docker images` | 查看本地镜像 |
| `docker run -p 8000:8000 my-image` | 运行容器 |
| `docker history my-image` | 查看镜像各层 |
