# Docker Compose 使用指南

## 在哪里使用

| 场景 | 说明 |
|------|------|
| 本地开发 | 一键启动包含数据库、缓存、应用服务在内的完整开发环境 |
| 测试环境 | 在测试服务器上部署多服务应用，保证环境一致性 |
| 演示环境 | 快速搭建包含多个依赖服务的演示环境 |

**前提条件**：已安装 Docker 和 Docker Compose。在终端中执行以下命令检查。

```bash
docker compose version
```

## 什么是 Docker Compose

Docker Compose 是一个编排工具，通过一个 YAML 文件定义多容器应用的所有服务、网络和卷，用一条命令完成全部服务的创建和启动。

**不使用 Docker Compose 时**：每个服务需要单独执行 `docker run`，手动管理端口映射、网络连通、环境变量，命令冗长且难以复用。

**使用 Docker Compose 后**：将所有服务的配置写在一个 `docker-compose.yml` 文件中，通过 `docker compose up -d` 一键启动所有服务。

## 使用步骤

Docker Compose 的使用需要两个东西：一个 `docker-compose.yml` 文件和安装了 Docker Compose 的环境。使用分三步：

### 第一步：编写 docker-compose.yml

在项目中创建 `docker-compose.yml` 文件，定义你的服务。

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
```

### 第二步：启动服务

在 `docker-compose.yml` 所在目录执行：

```bash
docker compose up -d
```

| 参数 | 说明 |
|------|------|
| `-d` | 后台运行（不加则在前台运行，日志直接输出到终端） |

### 第三步：查看服务状态

```bash
docker compose ps
```

## docker-compose.yml 文件结构

一个 `docker-compose.yml` 文件由三个顶级键组成：

| 键 | 是否必填 | 说明 |
|----|----------|------|
| `services` | 必填 | 定义应用的所有服务（每个服务是一个容器） |
| `networks` | 可选 | 定义服务间通信的网络 |
| `volumes` | 可选 | 定义持久化数据卷 |

### services（服务）

`services` 下每个子键是一个服务的名称，服务名即容器名的一部分。

```yaml
services:
  <服务名>:
    image: <镜像名:标签>      # 使用已有镜像
    build: <构建上下文路径>    # 或从 Dockerfile 构建
    ports:                    # 端口映射
      - "<宿主机端口>:<容器端口>"
    environment:              # 环境变量
      - KEY=VALUE
    volumes:                  # 数据卷挂载
      - <宿主机路径>:<容器路径>
    depends_on:               # 依赖的其他服务
      - <依赖的服务名>
```

### networks（网络）

```yaml
networks:
  <网络名>:
    driver: bridge            # 网络驱动，默认为 bridge
```

### volumes（数据卷）

```yaml
volumes:
  <卷名>:                     # 命名卷，Docker 自动管理存储路径
  <宿主机路径>:               # 或绑定挂载，指定宿主机具体路径
    driver: local
```

## 常用配置项

每个服务可以配置以下参数：

| 配置项 | 是否必填 | 说明 |
|--------|----------|------|
| `image` | 二选一 | 指定使用的镜像（与 `build` 二选一，填 `image` 则用已有镜像，填 `build` 则从 Dockerfile 构建） |
| `build` | 二选一 | 指定 Dockerfile 所在目录，启动时自动构建镜像 |
| `container_name` | 可选 | 指定容器名称（不指定则使用 `<项目名>_<服务名>_<序号>`） |
| `ports` | 按需 | 端口映射，格式 `"<宿主机端口>:<容器端口>"` |
| `expose` | 可选 | 暴露端口给其他服务，不映射到宿主机 |
| `environment` | 按需 | 设置环境变量，支持列表或字典格式 |
| `env_file` | 按需 | 从 `.env` 文件加载环境变量 |
| `volumes` | 按需 | 挂载数据卷或宿主机目录 |
| `depends_on` | 按需 | 声明服务依赖关系，控制启动顺序 |
| `networks` | 按需 | 指定服务加入的网络 |
| `restart` | 可选 | 重启策略，见下表 |
| `command` | 可选 | 覆盖镜像默认的启动命令 |
| `entrypoint` | 可选 | 覆盖镜像默认的 entrypoint |
| `healthcheck` | 可选 | 健康检查配置 |

### restart 重启策略

| 值 | 说明 |
|----|------|
| `no` | 不自动重启（默认） |
| `always` | 任何情况下都重启 |
| `on-failure` | 仅在退出码非零时重启 |
| `unless-stopped` | 始终重启，除非手动停止 |

## 常用命令

### 启动 / 停止 / 重启

```bash
# 启动所有服务（后台模式）
docker compose up -d

# 启动指定服务
docker compose up -d <服务名>

# 停止所有服务
docker compose down

# 重启所有服务
docker compose restart

# 重启指定服务
docker compose restart <服务名>
```

### 查看状态

```bash
# 查看运行中的服务
docker compose ps

# 查看服务日志
docker compose logs

# 实时跟踪日志
docker compose logs -f

# 查看指定服务的日志
docker compose logs <服务名>
```

### 构建镜像

```bash
# 构建所有服务的镜像
docker compose build

# 构建指定服务
docker compose build <服务名>

# 不使用缓存构建
docker compose build --no-cache
```

### 执行命令

```bash
# 在运行的容器中执行命令
docker compose exec <服务名> <命令>

# 进入容器的交互式终端
docker compose exec -it <服务名> /bin/bash
```

### 其他命令

```bash
# 查看版本
docker compose version

# 配置校验（检查 yml 文件语法是否正确）
docker compose config

# 暂停服务
docker compose pause

# 恢复服务
docker compose unpause

# 删除服务产生的数据卷（加 -v）
docker compose down -v
```

## 完整示例

### 示例一：WordPress + MySQL

```yaml
services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress-app
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress-data:/var/www/html
    depends_on:
      - mysql
    networks:
      - wp-network
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    container_name: mysql-db
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - wp-network
    restart: unless-stopped

networks:
  wp-network:
    driver: bridge

volumes:
  wordpress-data:
  mysql-data:
```

### 示例二：Spring Boot + Redis + MySQL

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: spring-boot-app
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/mydb
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: rootpassword
      SPRING_REDIS_HOST: redis
    depends_on:
      - mysql
      - redis
    networks:
      - app-network
    restart: on-failure

  mysql:
    image: mysql:8.0
    container_name: mysql-db
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: mydb
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: redis-cache
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
```

### 示例三：前端 + 后端 + Nginx 反向代理

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx-proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - frontend
      - backend
    networks:
      - proxy-network
    restart: unless-stopped

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: app-frontend
    expose:
      - "3000"
    networks:
      - proxy-network
    restart: on-failure

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: app-backend
    expose:
      - "8080"
    networks:
      - proxy-network
    restart: on-failure

networks:
  proxy-network:
    driver: bridge
```

## 最佳实践

### 1. 使用 .env 文件管理环境变量

创建 `.env` 文件，敏感信息不写入 `docker-compose.yml`。

`.env` 文件内容：

```
DB_PASSWORD=mysecretpassword
DB_USER=admin
```

`docker-compose.yml` 中引用：

```yaml
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_USER: ${DB_USER}
```

### 2. 使用 docker-compose.override.yml 区分环境

Docker Compose 会自动合并 `docker-compose.yml` 和 `docker-compose.override.yml`。

- `docker-compose.yml`：生产环境配置
- `docker-compose.override.yml`：开发环境覆盖配置（不提交到版本控制）

```yaml
# docker-compose.override.yml
services:
  db:
    ports:
      - "3306:3306"    # 仅开发环境暴露数据库端口
```

### 3. 使用 healthcheck 确保服务就绪

```yaml
services:
  mysql:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### 4. 使用命名卷持久化数据

```yaml
volumes:
  mysql-data:    # Docker 自动管理存储路径
```

不要将数据库数据直接挂载到宿主机固定路径，使用命名卷由 Docker 统一管理。

### 5. 指定镜像版本标签

```yaml
# 错误：使用 latest，环境不一致
image: nginx:latest

# 正确：指定具体版本
image: nginx:1.25-alpine
```

### 6. 使用 .dockerignore 排除不必要的文件

在项目根目录创建 `.dockerignore`：

```
.git
.gitignore
node_modules
*.log
.env
```

## 常见问题

### 服务启动顺序不对

`depends_on` 只保证容器启动顺序，不保证服务就绪。如果需要等待依赖服务完全就绪，使用 `healthcheck`。

```yaml
services:
  app:
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      retries: 5
```

### 容器之间无法通信

确保服务在同一个网络中。使用服务名作为主机名访问其他服务。

```yaml
# 应用连接数据库时，主机名写服务名 "mysql"，不是 "localhost"
SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/mydb
```

### 修改配置后不生效

```bash
# 重新创建容器
docker compose up -d --force-recreate

# 重新构建镜像
docker compose up -d --build
```
