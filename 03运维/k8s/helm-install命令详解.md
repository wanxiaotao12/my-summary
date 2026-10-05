# Helm Install 命令详解

## 一、基本语法

```bash
helm install <release-name> <chart-path> [flags]
```

| 参数 | 说明 |
|---|---|
| `<release-name>` | 给这次部署起一个名字，后续用它来升级、回滚、卸载 |
| `<chart-path>` | Chart 的位置，可以是本地目录、压缩包或远程仓库 |

## 二、最简示例

```bash
helm install my-nginx ./my-nginx
```

- `my-nginx` — release 名称
- `./my-nginx` — Chart 在当前目录的子文件夹中

这条命令使用 Chart 自带的默认配置，部署到 `default` 命名空间。

## 三、完整命令拆解

以项目中的实际命令为例（已调整为规范写法）：

```bash
helm install nvwa . -n aml-nvwa --create-namespace -f values.yaml
```

逐部分解释：

| 片段 | 含义 |
|---|---|
| `helm` | 调用 Helm CLI |
| `install` | 执行安装操作 |
| `nvwa` | release 名称 |
| `.` | Chart 路径为当前目录 |
| `-n aml-nvwa` | 指定目标命名空间为 `aml-nvwa` |
| `--create-namespace` | 如果 `aml-nvwa` 不存在，自动创建它 |
| `-f values.yaml` | 使用当前目录下的 `values.yaml` 覆盖默认配置 |

## 四、常用参数（Flags）

### 4.1 命名空间相关

```bash
# 指定命名空间
helm install myapp ./myapp -n my-namespace

# 命名空间不存在时自动创建
helm install myapp ./myapp -n my-namespace --create-namespace
```

### 4.2 配置文件相关

```bash
# 使用自定义 values 文件
helm install myapp ./myapp -f my-values.yaml

# 使用多个 values 文件（后面的覆盖前面的）
helm install myapp ./myapp -f base.yaml -f override.yaml

# 直接在命令行设置单个值
helm install myapp ./myapp --set image.tag=v1.2.3

# 设置多个值
helm install myapp ./myapp --set replicas=3 --set service.type=LoadBalancer
```

`-f` 和 `--set` 可以组合使用，`--set` 的优先级高于 `-f`。

### 4.3 部署行为相关

```bash
# 只生成 YAML 看看效果，不实际部署
helm install myapp ./myapp --dry-run

# 只生成 YAML 并输出到文件
helm template myapp ./myapp > output.yaml

# 等待所有资源就绪后再返回（默认等 5 分钟）
helm install myapp ./myapp --wait

# 自定义等待时间
helm install myapp ./myapp --wait --timeout 300s

# 部署时不验证 webhook
helm install myapp ./myapp --no-hooks
```

### 4.4 版本相关

```bash
# 指定 Chart 版本
helm install myapp ./myapp --version 1.2.0
```

## 五、安装后的管理命令

```bash
# 查看所有 release
helm list
helm list -n aml-nvwa          # 指定命名空间
helm list -A                    # 所有命名空间

# 查看 release 状态
helm status nvwa

# 查看部署历史
helm history nvwa

# 升级
helm upgrade nvwa . -f values.yaml

# 安装或升级（不存在则安装，已存在则升级）
helm upgrade --install nvwa . -f values.yaml

# 回滚到指定版本
helm rollback nvwa 1

# 卸载
helm uninstall nvwa -n aml-nvwa
```

## 六、两种写法的对比

```bash
# 写法一：最简形式（示例中的写法）
helm install my-nginx ./my-nginx

# 写法二：完整形式（项目 deploy.sh 中的写法，已调整为规范写法）
helm install nvwa . -n aml-nvwa --create-namespace -f values.yaml
```

| 区别 | 写法一 | 写法二 |
|---|---|---|
| 命名空间 | 默认 `default` | 指定 `aml-nvwa` |
| 自动创建命名空间 | 否 | 是（`--create-namespace`） |
| 配置文件 | 使用 Chart 默认值 | 使用自定义 `values.yaml` |
| Chart 路径 | 子目录 `./my-nginx` | 当前目录 `.` |

两条命令的核心都是 `helm install <release-name> <chart-path>`，其余参数只是控制部署到哪个命名空间、用什么配置。

## 七、常见错误

```bash
# 错误：release 名称已存在
# Error: cannot re-use a name that is still in use
helm install myapp ./myapp
# 解决：换个名字，或用 helm upgrade
helm upgrade myapp ./myapp

# 错误：命名空间不存在
# Error: create: namespaces "xxx" not found
helm install myapp ./myapp -n xxx
# 解决：加上 --create-namespace，或先手动创建
kubectl create namespace xxx

# 错误：Chart 路径不对
# Error: path "xxx" not found
helm install myapp ./xxx
# 解决：检查路径，确保目录包含 Chart.yaml
```
