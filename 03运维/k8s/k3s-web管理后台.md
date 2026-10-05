# K3s Web 管理后台

K3s 支持多种 Web 管理界面，最推荐的是 Rancher。

## 1. Rancher（推荐）

Rancher 是 K3s 母公司 Rancher Labs 开发的企业级 Kubernetes 管理平台，与 K3s 深度集成。

### 特性

- **多集群管理**：同时管理多个 K3s/K8s 集群
- **可视化操作**：图形化界面创建、编辑、删除资源
- **用户权限**：细粒度的 RBAC 权限控制
- **应用商店**：内置 Helm Chart 应用市场
- **监控告警**：集成 Prometheus + Grafana 监控
- **日志管理**：集中式日志收集与查看
- **CI/CD**：内置 Fleet 实现 GitOps 持续部署

### 安装 Rancher

```bash
# 使用 Helm 安装（推荐）
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

# 创建命名空间
kubectl create namespace cattle-system

# 安装 cert-manager（用于 TLS 证书）
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace

# 安装 Rancher
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.your-domain.com \
  --set bootstrapPassword=admin-password
```

### 访问 Rancher

浏览器打开 `https://rancher.your-domain.com`，使用 bootstrapPassword 登录。

### Rancher 界面功能

- **集群概览**：查看所有集群状态、节点、资源使用情况
- **工作负载**：管理 Deployment、StatefulSet、DaemonSet、Job
- **服务发现**：管理 Service、Ingress
- **配置**：管理 ConfigMap、Secret
- **存储**：管理 PersistentVolume、StorageClass
- **命名空间**：管理 Namespace 和 ResourceQuota
- **用户认证**：集成 LDAP、AD、OAuth 等

---

## 2. Kubernetes Dashboard（官方）

Kubernetes 官方的 Web UI，适用于所有 K8s 集群包括 K3s。

### 安装

```bash
# 部署 Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.1/aio/deploy/recommended.yaml

# 创建访问用户
kubectl create serviceaccount admin-user -n kubernetes-dashboard

# 绑定集群管理员权限
kubectl create clusterrolebinding admin-user \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:admin-user

# 获取访问 Token
kubectl -n kubernetes-dashboard create token admin-user
```

### 访问 Dashboard

```bash
# 启动代理
kubectl proxy

# 浏览器访问
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

### 功能

- 查看集群状态和资源
- 管理工作负载（Deployment、Pod 等）
- 查看日志
- 编辑资源 YAML
- 查看事件

---

## 3. KubeSphere

KubeSphere 是青云科技开发的 K8s 管理平台，功能丰富且界面友好。

### 安装

```bash
# 安装 KubeSphere
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.4.1/kubesphere-installer.yaml
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.4.1/cluster-configuration.yaml

# 查看安装进度
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
```

### 访问

默认账号：`admin`  
默认密码：`P@88w0rd`

### 特性

- 多集群管理
- DevOps 流水线
- 应用商店
- 监控告警
- 日志审计
- 服务网格

---

## 4. Lens（桌面应用）

Lens 是一个 Kubernetes IDE，虽然不是纯 Web UI，但提供类似体验。

### 安装

```bash
# macOS
brew install --cask lens

# 或从官网下载
# https://k8slens.dev/
```

### 特性

- 实时集群视图
- 资源拓扑图
- 日志查看
- Terminal 集成
- 性能监控
- 多集群管理

---

## 对比总结

| 特性 | Rancher | K8s Dashboard | KubeSphere | Lens |
|------|---------|---------------|------------|------|
| 部署难度 | 中等 | 简单 | 中等 | 桌面应用 |
| 多集群管理 | ✅ 强 | ❌ 单集群 | ✅ 强 | ✅ 强 |
| 用户权限 | ✅ 细粒度 | ⚠️ 基础 | ✅ 细粒度 | ⚠️ 基础 |
| 应用商店 | ✅ | ❌ | ✅ | ❌ |
| 监控告警 | ✅ 集成 | ❌ | ✅ 集成 | ✅ 基础 |
| 日志管理 | ✅ 集成 | ⚠️ 基础 | ✅ 集成 | ⚠️ 基础 |
| CI/CD | ✅ Fleet | ❌ | ✅ | ❌ |
| 适用场景 | 生产环境 | 开发测试 | 企业级 | 个人开发 |

---

## 推荐方案

### 开发测试环境

**Kubernetes Dashboard**：轻量、快速、够用

### 生产环境

**Rancher**：与 K3s 深度集成，功能全面，社区活跃

### 企业级需求

**KubeSphere**：功能最丰富，适合大型团队和复杂场景

### 个人开发

**Lens**：桌面应用，体验流畅，适合单人使用

---

## 快速验证（Rancher）

```bash
# 在 K3s 集群上快速安装 Rancher（单节点测试）
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

kubectl create namespace cattle-system

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.local \
  --set replicas=1 \
  --set auditLog.level=0

# 等待就绪
kubectl -n cattle-system rollout status deploy/rancher

# 获取初始密码
kubectl -n cattle-system get secret bootstrap-secret -o jsonpath='{.data.bootstrapPassword}' | base64 -d

# 本地测试：添加 hosts 映射
echo "<K3s-IP> rancher.local" | sudo tee -a /etc/hosts

# 访问 https://rancher.local
```
