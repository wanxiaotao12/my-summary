# K3s 与 K8s 的区别

## 一句话总结

K3s 是轻量级的 Kubernetes，"删减版但够用的 K8s"，在保留核心编排能力的同时，用精简和集成换来了轻量与便捷。

## 核心对比

| 维度 | K8s (Kubernetes) | K3s |
|------|------------------|-----|
| 安装复杂度 | 高，需手动搭建 etcd、API Server、Controller 等多个组件 | 低，单条命令即可安装，所有组件打包在一个二进制文件中 |
| 资源占用 | 高，生产环境通常需要多个节点，每个节点至少 2GB+ 内存 | 低，单节点最低 512MB 内存即可运行 |
| 组件数量 | 完整组件：etcd、kube-apiserver、kube-scheduler、kube-controller-manager、kubelet、kube-proxy 等 | 精简组件：移除了云厂商插件、Docker（默认用 containerd）、Ingress controller 替换为 Traefik 等 |
| 默认存储 | 无内置默认存储 | 内置 SQLite 作为默认数据存储（也可切换为 etcd3/MySQL/PostgreSQL） |
| 网络插件 | 需自行选择安装（Calico、Flannel、Cilium 等） | 默认内置 Flannel |
| 容器运行时 | 支持 Docker、containerd、CRI-O 等 | 默认使用 containerd |
| 适用场景 | 大规模生产环境、多云/混合云部署 | 边缘计算、IoT、开发测试环境、资源受限场景、学习 K8s |
| 二进制大小 | 多个独立二进制，总体较大 | 单一二进制文件，约 60-100MB |
| 证书管理 | 需手动或用工具管理 | 自动管理证书 |
| 社区支持 | CNCF 顶级项目，社区庞大 | Rancher 主导开发，CNCF 毕业项目 |

## K3s 精简了什么

1. **移除了过时/非必要的 API 版本**：去掉了 alpha/beta 阶段的 API
2. **移除了内置的云厂商提供商代码**：不依赖特定云平台
3. **默认使用 containerd**：不再依赖 Docker，减少运行时开销
4. **内置轻量级存储后端**：默认 SQLite，替代 etcd
5. **内置 Traefik 作为 Ingress Controller**：开箱即用
6. **内置 CoreDNS、Local Path Provisioner、ServiceLB**：常用组件打包
7. **单一二进制 + 单一进程**：所有控制平面组件打包在一起

## K3s 保留了什么

- 完整的 Kubernetes API（去除过时版本后）
- 核心编排能力：Pod、Deployment、Service、ConfigMap、Secret 等
- RBAC、Namespace、CRD
- kubectl 完全兼容
- Helm 完全兼容
- 多节点集群能力
- 自动证书管理与轮换

## K3s 集群部署

K3s 完全支持多节点集群部署，可以搭建高可用集群。

### 单节点安装（开发测试）

```bash
# 安装 server 节点（控制平面 + 工作节点）
curl -sfL https://get.k3s.io | sh -
```

### 多节点集群（生产环境）

```bash
# 第一个 server 节点（控制平面）
curl -sfL https://get.k3s.io | sh -

# 获取 token（用于其他节点加入）
cat /var/lib/rancher/k3s/server/node-token

# 添加更多 server 节点（高可用控制平面）
curl -sfL https://get.k3s.io | K3S_TOKEN=<token> sh -s server --server https://<first-server-ip>:6443

# 添加 agent 节点（工作节点）
curl -sfL https://get.k3s.io | K3S_TOKEN=<token> K3S_URL=https://<server-ip>:6443 sh -
```

### 高可用架构

- **3 个 server 节点**：推荐的最小高可用配置
- **外部数据库**：生产环境建议使用外部 etcd、MySQL 或 PostgreSQL
- **负载均衡**：在多个 server 节点前配置负载均衡器

## 典型使用场景

- **开发测试**：本地快速搭建 K8s 环境
- **边缘计算 / IoT**：资源受限的设备
- **CI/CD**：轻量级集群用于流水线测试
- **学习 K8s**：比 Minikube 更接近真实环境
- **小型生产环境**：对资源敏感的小规模部署

## 总结

K3s 不是"阉割版"，而是"精简版"。它去掉了 K8s 中企业级但非核心的部分，保留了 90% 以上日常使用的功能，换来了更低的资源消耗和更简单的运维体验。对于中小规模场景和学习用途，K3s 是更实用的选择。
