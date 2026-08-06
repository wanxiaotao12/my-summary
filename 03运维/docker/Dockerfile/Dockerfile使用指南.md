# Dockerfile 使用指南

## 在哪里使用

| 环境 | 说明 |
|------|------|
| 本地电脑 | 在开发者的电脑上安装 Docker Desktop，用于本地开发和测试 |
| 服务器 | 在 Linux 服务器上安装 Docker，用于部署应用 |
| CI/CD 流水线 | 在 Jenkins、GitLab CI 等持续集成工具中自动构建镜像 |

**前提条件**：无论在哪种环境，都需要先安装 Docker。安装完成后，在终端中执行以下命令。

## 使用场景

Dockerfile 的使用需要在**安装了 Docker 的环境**中执行。写好 Dockerfile 后，怎么把它跑起来分两种场景：

| 场景 | 工具 | 适用情况 |
|------|------|----------|
| 场景一：单容器 | `docker` 命令 | 只跑一个应用（如单个 web 服务），用 `docker build` + `docker run` |
| 场景二：多容器编排 | `docker-compose` | 一次跑多个服务（如 web + 数据库），用一个 yml 文件统一管理 |

两种场景共用同一份 Dockerfile，区别只在"怎么把它启动起来"。下面分别说明。

## 场景一：单容器

只有一个应用要跑时，手工用 `docker` 命令操作，使用分三步：

### 第一步：构建镜像

命令格式（方括号 `[]` 内为可选，其余为必填）：

```
docker build [-f <Dockerfile文件>] -t <镜像名>:<标签> <构建上下文路径>
```

| 组成部分 | 是否必填 | 说明 |
|----------|----------|------|
| `-f <Dockerfile文件>` | 可选 | `--file`（指定文件）的缩写。默认使用当前目录下名为 `Dockerfile` 的文件；当文件名不叫 `Dockerfile`（如 `Dockerfile.prod`）时，用它指定 |
| `-t` | 强烈建议填 | `--tag`（打标签）的缩写，作用是给镜像起名字和版本号，方便后续用名字引用；不填则镜像无名字，只能用一长串 ID 引用 |
| `<镜像名>:<标签>` | 强烈建议填 | `-t` 后面跟的内容，如 `my-app:v1.0`，即镜像名为 my-app、标签为 v1.0 |
| `<构建上下文路径>` | 必填 | Dockerfile 及所需文件所在的目录，`.` 表示当前目录 |

**填写顺序**：先想好镜像叫什么名字（`镜像名:标签`），再指定 Dockerfile 所在目录（`构建上下文路径`）；若文件名不是默认的 `Dockerfile`，再加 `-f` 指定。

**指定 Dockerfile 文件的示例**（文件名不叫 `Dockerfile` 时用 `-f`）：

```bash
docker build -f Dockerfile.prod -t my-app:prod .
#            ↑ 指定用哪个文件     ↑ 镜像名       ↑ 上下文
```

在终端中，进入 Dockerfile 所在的目录，执行：

```bash
docker build -t my-app:v1.0 .
```

| 参数 | 说明 |
|------|------|
| `-t my-app:v1.0` | 给镜像命名为 my-app，标签为 v1.0 |
| `.` | 构建上下文路径，`.` 表示 Dockerfile 所在的当前目录 |

### 第二步：查看镜像

```bash
docker images
```

### 第三步：运行容器

```bash
docker run -p 8000:8000 my-app:v1.0
```

| 参数 | 是否必填 | 说明 |
|------|----------|------|
| `-p 8000:8000` | 按需 | 端口映射，左边是宿主机端口，右边是容器端口 |
| `my-app:v1.0` | 必填 | 要运行的镜像名和标签 |

除上面这条命令用到的参数外，还有几个**常用可选参数**（按需添加）：

| 可选参数 | 说明 |
|----------|------|
| `-d` | 后台运行（不加则在前台运行，占用当前终端） |
| `--name my-container` | 给容器起个名字，方便后续用名字操作 |

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

## 场景二：多容器编排（docker-compose）

当应用由多个服务组成（比如一个 web 服务 + 一个数据库），用上面手工 `docker run` 的方式会很麻烦——要手动起多个容器、配置它们之间的网络和依赖。**docker-compose** 用一个 `docker-compose.yml` 文件描述这些服务，一条命令就能把所有服务一起拉起来。

### 前提条件

除了安装 Docker，还需要安装 docker-compose。新版本 Docker Desktop 已内置 `docker compose` 命令（注意是空格，不是连字符）；老版本需单独安装 `docker-compose`。下文命令均以 `docker compose` 为例。

### 使用步骤

#### 第一步：编写 docker-compose.yml

在 Dockerfile 所在目录新建一个 `docker-compose.yml`，描述要跑哪些服务。下面是一个 **web 应用 + MySQL 数据库** 的最小示例：

```yaml
services:
  web:                        # 第一个服务：你的应用
    build: .                  # 用当前目录的 Dockerfile 构建镜像
    ports:
      - "8000:8000"           # 端口映射：宿主机端口:容器端口
    depends_on:
      - db                    # 表示 web 依赖 db，compose 会先启动 db

  db:                         # 第二个服务：数据库（直接用现成镜像，无需 Dockerfile）
    image: mysql:8.0          # 直接拉取官方 MySQL 镜像
    environment:
      MYSQL_ROOT_PASSWORD: 123456   # 设置数据库密码
    volumes:
      - db_data:/var/lib/mysql     # 数据持久化：数据存在名为 db_data 的卷里

volumes:
  db_data:                    # 声明上面用到的数据卷
```

#### 关键字段说明

| 字段 | 作用 |
|------|------|
| `services` | 列出所有要启动的服务，每个服务是一个独立容器 |
| `build` | 说明这个服务用哪个 Dockerfile 构建（`.` 表示当前目录） |
| `image` | 直接用现成镜像启动，不构建（如数据库这类现成组件） |
| `ports` | 端口映射，格式 `宿主机端口:容器端口` |
| `depends_on` | 声明依赖，被依赖的服务先启动 |
| `environment` | 给容器传环境变量（如数据库密码） |
| `volumes` | 数据卷，用于数据持久化，容器删了数据还在 |

> 说明：`build` 和 `image` 二选一——`build` 用你的 Dockerfile 打包自己的应用，`image` 直接用官方现成镜像（数据库、缓存这类组件通常这么做，不必自己写 Dockerfile）。

#### 第二步：启动所有服务

在 `docker-compose.yml` 所在目录执行：

```bash
docker compose up -d
#          ↑ -d 表示后台运行，不加会占用当前终端
```

compose 会自动构建 web 的镜像、拉取 MySQL 镜像，分别启动两个容器，并把它们连到同一个网络里。

#### 第三步：查看与停止

```bash
# 查看正在运行的服务
docker compose ps

# 查看某个服务的日志（如 web）
docker compose logs web

# 停止并移除所有服务、网络（数据卷保留）
docker compose down
```

| 命令 | 作用 |
|------|------|
| `docker compose up -d` | 构建并后台启动所有服务 |
| `docker compose ps` | 查看各服务容器状态 |
| `docker compose logs <服务名>` | 查看某服务日志 |
| `docker compose down` | 停止并移除所有服务（数据卷不删） |

## 完整示例

以下是三种常见语言应用的 Dockerfile 文件完整示例，展示如何将指令组合写进一个 Dockerfile 文件中。

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

以下是一个 Java 应用的 Dockerfile 文件内容，使用了多阶段构建（两个 FROM，详见 [Dockerfile常用指令指南.md](./Dockerfile常用指令指南.md) 的「多阶段构建」一节）：

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

> 说明：Maven 默认打包出的 jar 名是 `<artifactId>-<版本>.jar`（如 `myapp-1.0.jar`）。示例里用 `app.jar` 是为了简洁，实际项目中需在 `pom.xml` 里配置 `<build><finalName>app</finalName></build>`，产物才会叫 `app.jar`。

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