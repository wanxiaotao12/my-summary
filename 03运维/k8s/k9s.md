# k9s - Kubernetes 集群管理工具

## 一句话介绍

k9s 是一个基于终端的 Kubernetes 集群管理 UI，提供类似 vim 的交互方式，让你在命令行中快速查看、管理和调试 K8s 资源。

## 核心特性

- **实时视图**：实时展示集群中的 Pod、Deployment、Service、Node 等资源状态
- **快速导航**：类似 vim 的快捷键操作，无需记忆复杂的 kubectl 命令
- **日志查看**：直接在终端中查看 Pod 日志，支持实时流式输出
- **Shell 访问**：一键进入 Pod 的 shell 环境
- **资源编辑**：直接在终端中编辑 K8s 资源 YAML
- **多集群支持**：支持多个 kubeconfig 上下文切换
- **插件系统**：支持自定义插件扩展功能
- **资源过滤**：强大的过滤和搜索功能

## 安装

### macOS

```bash
# 使用 Homebrew
brew install k9s

# 或使用 MacPorts
sudo port install k9s
```

### Linux

```bash
# 下载二进制文件
curl -sS https://webi.sh/k9s | sh

# 或从 GitHub Releases 下载
wget https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
tar -xzf k9s_Linux_amd64.tar.gz
sudo mv k9s /usr/local/bin/
```

### Windows

```bash
# 使用 Chocolatey
choco install k9s

# 使用 Scoop
scoop install k9s
```

## 基本使用

### 启动 k9s

```bash
# 使用默认 kubeconfig 上下文
k9s

# 指定上下文
k9s --context <context-name>

# 指定命名空间
k9s -n <namespace>

# 只读模式
k9s --readonly
```

### 常用快捷键

| 快捷键 | 功能 |
|--------|------|
| `:pods` 或 `:po` | 查看 Pod 列表 |
| `:deployments` 或 `:deploy` | 查看 Deployment 列表 |
| `:services` 或 `:svc` | 查看 Service 列表 |
| `:nodes` 或 `:no` | 查看 Node 列表 |
| `:namespaces` 或 `:ns` | 查看 Namespace 列表 |
| `:all` | 查看所有资源 |
| `:ctx` | 切换 kubeconfig 上下文 |
| `:q` | 退出 k9s |
| `/` | 搜索/过滤 |
| `d` | 删除选中的资源 |
| `e` | 编辑选中的资源 |
| `l` | 查看 Pod 日志 |
| `s` | 进入 Pod shell |
| `y` | 查看资源 YAML |
| `w` | 查看资源事件 |
| `Ctrl+D` | 向下滚动日志 |
| `Ctrl+U` | 向上滚动日志 |

### 典型工作流

#### 1. 查看 Pod 状态

```bash
# 启动 k9s
k9s

# 输入 :po 查看 Pod 列表
# 使用 / 过滤特定 Pod
# 按 l 查看日志
# 按 s 进入 shell
```

#### 2. 排查问题

```bash
# 查看有问题的 Pod
:po
/CrashLoop  # 过滤包含 CrashLoop 的 Pod

# 查看日志
l

# 查看事件
w

# 编辑配置
e
```

#### 3. 切换集群

```bash
# 切换上下文
:ctx

# 选择目标集群
# 使用方向键选择，回车确认
```

## 配置文件

k9s 的配置文件位于 `~/.config/k9s/config.yaml`：

```yaml
k9s:
  refreshRate: 2          # 刷新频率（秒）
  maxConnRetry: 5         # 最大连接重试次数
  enableMouse: false      # 是否启用鼠标
  headless: false         # 是否无头模式
  readOnly: false         # 是否只读模式
  ui:
    skin: default         # 主题
    defaultsView: pods    # 默认视图
  logs:
    tail: 100             # 默认日志行数
    buffer: 5000          # 日志缓冲区大小
```

## 自定义皮肤

k9s 支持自定义主题，在 `~/.config/k9s/skins/` 目录下创建 YAML 文件：

```yaml
# ~/.config/k9s/skins/my-skin.yaml
k9s:
  body:
    fgColor: white
    bgColor: black
  info:
    fgColor: cyan
  dialog:
    fgColor: white
    bgColor: black
```

然后在配置文件中引用：

```yaml
k9s:
  ui:
    skin: my-skin
```

## 插件示例

在 `~/.config/k9s/plugins.yaml` 中定义插件：

```yaml
plugins:
  # 查看 Pod 的资源使用情况
  top:
    shortCut: Shift-T
    description: "View pod resources"
    scopes:
      - pods
    command: kubectl
    args:
      - top
      - pod
      - -n
      - $NAMESPACE
      - --sort-by=cpu
```

## 与 kubectl 的对比

| 功能 | kubectl | k9s |
|------|---------|-----|
| 查看资源 | `kubectl get pods` | `:po` |
| 查看日志 | `kubectl logs -f <pod>` | 选中 Pod 后按 `l` |
| 进入 Shell | `kubectl exec -it <pod> -- sh` | 选中 Pod 后按 `s` |
| 编辑资源 | `kubectl edit <resource>` | 选中资源后按 `e` |
| 删除资源 | `kubectl delete <resource>` | 选中资源后按 `d` |
| 切换上下文 | `kubectl config use-context` | `:ctx` |
| 学习曲线 | 需要记忆大量命令 | 类似 vim，上手快 |

## 适用场景

- **日常开发**：快速查看 Pod 状态和日志
- **问题排查**：实时观察集群状态，快速定位问题
- **演示教学**：直观展示 K8s 资源关系
- **运维监控**：轻量级的集群监控工具

## 总结

k9s 是 Kubernetes 管理者的瑞士军刀，将复杂的 kubectl 命令转化为直观的终端 UI，大幅提升日常 K8s 操作效率。对于需要频繁与 K8s 集群交互的开发者、运维人员来说，k9s 是必备工具。
