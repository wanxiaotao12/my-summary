# Helm 入门指南

## 一、Helm 是什么

Helm 是 Kubernetes 的包管理工具，类似于 Linux 的 `apt`（Debian）或 `yum`（CentOS）。

它的作用是**批量生成和管理 Kubernetes 的 YAML 配置文件**，让你用一条命令就能部署复杂的应用。

### 1.1 没有 Helm 的时候

部署一个应用到 Kubernetes，你需要手动创建一堆 YAML 文件：

```bash
# 方式一：逐个应用
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f pvc.yaml

# 方式二：一次性应用整个目录
kubectl apply -f ./manifests/

# 方式三：使用 Kustomize（Kubernetes 原生方案）
kubectl apply -k ./overlays/production/
```

虽然 `kubectl` 支持批量应用，但仍然存在问题：
- **没有模板化** - 不同环境（开发/测试/生产）需要维护多套完整的 YAML，难以复用
- **没有参数化** - 修改镜像版本、副本数等需要直接改 YAML 文件
- **没有版本管理** - 缺乏统一的升级、回滚、状态追踪机制
- **没有依赖管理** - 应用依赖的子组件（如数据库）需要手动协调

### 1.2 有了 Helm 之后

```bash
# 一条命令部署所有资源
helm install myapp ./myapp-chart

# 一条命令升级
helm upgrade myapp ./myapp-chart

# 一条命令回滚
helm rollback myapp 1

# 一条命令卸载
helm uninstall myapp
```

### 1.3 类比理解

| 工具 | 平台 | 包格式 | 安装命令 |
|---|---|---|---|
| apt | Debian/Ubuntu | `.deb` | `apt install nginx` |
| npm | Node.js | `package.json` | `npm install express` |
| **Helm** | **Kubernetes** | **Chart** | **`helm install myapp ./chart`** |

## 二、核心概念

| 概念 | 说明 |
|---|---|
| **Chart** | 一个包含所有 Kubernetes 资源模板的文件包，就是"安装包" |
| **Release** | Chart 部署到集群后的一个实例，就是"安装后的应用" |
| **Repository** | 存放和分享 Chart 的仓库，就是"应用商店" |

## 三、Chart 的目录结构

### 3.1 基础 Chart 结构（单个应用）

```
myapp/
├── Chart.yaml          # Chart 的元信息（名称、版本、描述）
├── values.yaml         # 默认配置参数
├── templates/          # Kubernetes 资源模板文件
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── ...
└── charts/             # 依赖的其他 Chart（可选）
```

**说明：**
- 一个 Chart 对应一个应用
- 多环境通过不同的 values 文件区分（`values-dev.yaml`、`values-prod.yaml`）

### 3.2 多环境支持

```
myapp/
├── Chart.yaml
├── values.yaml              # 默认配置（所有环境共享）
├── values-dev.yaml          # 开发环境特有配置
├── values-test.yaml         # 测试环境特有配置
├── values-prod.yaml         # 生产环境特有配置
└── templates/
    └── ...
```

部署时指定环境配置文件：

```bash
# 开发环境
helm install myapp-dev ./myapp -f values-dev.yaml

# 生产环境
helm install myapp-prod ./myapp -f values-prod.yaml
```

### 3.3 复杂应用 Chart（包含子组件）

如果一个应用包含多个子组件（如 Web + 数据库 + 缓存），可以使用**子 Chart（subchart）**：

```
myapp/
├── Chart.yaml              # 父 Chart 元信息
├── values.yaml             # 父 Chart 默认配置
├── charts/                 # 子 Chart 目录
│   ├── web/                # Web 应用子 Chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   ├── database/           # 数据库子 Chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   └── cache/              # 缓存子 Chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
└── templates/              # 父 Chart 自己的模板（可选）
```

**部署时可以通过父 Chart 的 values.yaml 统一配置所有子组件：**

```yaml
# myapp/values.yaml
web:
  replicaCount: 2
  image:
    tag: "1.0"

database:
  enabled: true
  type: mysql

cache:
  enabled: true
  type: redis
```

**配置优先级：**

子 Chart 保留自己的 `values.yaml` 作为默认配置，父 Chart 的 `values.yaml` 可以覆盖子 Chart 的配置：

```
配置优先级（从高到低）：
1. 命令行 -f 参数指定的 values 文件
2. 父 Chart 的 values.yaml
3. 子 Chart 自己的 values.yaml（默认值）
```

**示例：**

```yaml
# charts/web/values.yaml（子 Chart 默认配置）
replicaCount: 1
image:
  tag: "latest"

# myapp/values.yaml（父 Chart 覆盖子 Chart 配置）
web:
  replicaCount: 2        # 覆盖子 Chart 的 replicaCount
  image:
    tag: "1.0"           # 覆盖子 Chart 的 image.tag
```

最终生效的值：`replicaCount: 2`，`image.tag: "1.0"`

一条命令部署整个应用栈：

```bash
helm install myapp ./myapp
```

### 3.4 多环境 + 子 Chart 的组合

多环境配置和子 Chart 结合使用时，**在父 Chart 的环境配置文件中覆盖子 Chart 的配置**：

```
myapp/
├── Chart.yaml
├── values.yaml              # 默认配置
├── values-dev.yaml          # 开发环境
├── values-prod.yaml         # 生产环境
├── charts/
│   ├── web/
│   │   └── values.yaml      # 子 Chart 默认配置
│   └── database/
│       └── values.yaml      # 子 Chart 默认配置
└── templates/
```

**环境配置文件示例：**

```yaml
# values-dev.yaml（开发环境）
web:
  replicaCount: 1
  image:
    tag: "dev-1.0"

database:
  enabled: false             # 开发环境用外部数据库，不部署子 Chart
```

```yaml
# values-prod.yaml（生产环境）
web:
  replicaCount: 3
  image:
    tag: "1.0"

database:
  enabled: true              # 生产环境部署数据库子 Chart
  type: mysql
  storage: 100Gi
```

**部署：**

```bash
# 开发环境
helm install myapp-dev ./myapp -f values-dev.yaml

# 生产环境
helm install myapp-prod ./myapp -f values-prod.yaml
```

**完整优先级（从高到低）：**

1. `values-dev.yaml` 或 `values-prod.yaml`（命令行 `-f` 指定）
2. 父 Chart 的 `values.yaml`
3. 子 Chart 的 `values.yaml`（默认值）

## 四、Helm 的工作原理

```
helm install myapp ./myapp-chart
          │
          ▼
┌──────────────────────────────┐
│  Helm 做的事:                 │
│                              │
│  1. 读取 values.yaml 的参数   │
│  2. 把参数填入 templates/ 模板 │
│  3. 生成最终的 YAML           │
│  4. 提交给 Kubernetes        │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Kubernetes 做的事:           │
│                              │
│  1. 收到 YAML 配置            │
│  2. 根据 image 字段拉取镜像   │
│  3. 创建 Pod，启动容器        │
│  4. 创建 Service、PVC 等资源  │
└──────────────────────────────┘
```

**关键点：Helm 本身不拉取镜像、不创建容器。** 它只是生成 YAML 并提交给 Kubernetes，由 Kubernetes 负责实际的资源创建和镜像拉取。

## 五、入门示例：部署一个 Nginx

### 5.1 创建 Chart

```bash
helm create my-nginx
```

生成的目录结构：

```
my-nginx/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   └── ...
└── charts/
```

### 5.2 修改 values.yaml

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 80
```

### 5.3 模板是怎么工作的

`templates/deployment.yaml` 中写的是模板语法：

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

Helm 用 `values.yaml` 中的值替换模板变量，生成最终 YAML：

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: my-nginx
          image: "nginx:1.25"
          ports:
            - containerPort: 80
```

### 5.4 部署

```bash
# 安装
helm install my-nginx ./my-nginx

# 查看已安装的 release
helm list

# 查看生成的 Kubernetes 资源
kubectl get all -l app.kubernetes.io/name=my-nginx

# 升级（比如改副本数为 3）
# 先修改 values.yaml 中 replicaCount: 3
helm upgrade my-nginx ./my-nginx

# 查看历史版本
helm history my-nginx

# 回滚到上一个版本
helm rollback my-nginx 1

# 卸载
helm uninstall my-nginx
```

### 5.5 不同环境使用不同配置

```bash
# 开发环境
helm install my-nginx ./my-nginx -f values-dev.yaml

# 生产环境
helm install my-nginx ./my-nginx -f values-prod.yaml
```

`values-prod.yaml` 只需要写和默认值不同的部分：

```yaml
replicaCount: 5

image:
  tag: "1.25"

service:
  type: LoadBalancer
  port: 80
```

## 六、常用命令速查

```bash
# 创建 Chart
helm create <chart-name>

# 安装
helm install <release-name> <chart-path>

# 升级
helm upgrade <release-name> <chart-path>

# 安装或升级（不存在则安装，存在则升级）
helm upgrade --install <release-name> <chart-path>

# 卸载
helm uninstall <release-name>

# 查看已安装的 release
helm list

# 查看某个 release 的状态
helm status <release-name>

# 查看发布历史
helm history <release-name>

# 回滚
helm rollback <release-name> <revision>

# 预览生成的 YAML（不实际部署）
helm template <release-name> <chart-path>

# 打包 Chart
helm package <chart-path>
```

## 七、总结

| 问题 | 答案 |
|---|---|
| Helm 是什么？ | Kubernetes 的包管理工具 |
| Chart 是什么？ | 一组 Kubernetes YAML 模板的集合 |
| Helm 做了什么？ | 读取配置 + 填充模板 = 生成 YAML → 提交给 K8s |
| Helm 拉镜像吗？ | 不拉，Kubernetes 根据 YAML 中的 image 字段拉取 |
| 为什么不直接写 YAML？ | 模板化复用、参数化配置、一键安装/升级/回滚 |
