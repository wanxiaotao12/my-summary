# Grafana - 可视化监控平台

## 一句话介绍

Grafana 是一个开源的数据可视化和监控平台，用于展示时间序列数据（指标、日志、追踪），是 Kubernetes 生态中最常用的监控可视化工具。

## 核心功能

- **数据可视化**：丰富的图表类型（折线图、柱状图、仪表盘、热力图等）
- **多数据源支持**：Prometheus、InfluxDB、Elasticsearch、MySQL、PostgreSQL、Loki 等
- **告警通知**：支持邮件、Slack、钉钉、企业微信等告警渠道
- **仪表盘模板**：社区提供大量现成的 Dashboard 模板
- **日志分析**：集成 Loki 实现日志查询和分析
- **追踪分析**：支持分布式追踪（Jaeger、Zipkin）
- **权限管理**：细粒度的用户和团队权限控制
- **插件生态**：丰富的插件扩展功能

## 在 K8s 中的角色

```
┌─────────────────────────────────────────────────────┐
│                    K8s 集群                          │
│                                                      │
│  ┌──────────┐      ┌──────────────┐                 │
│  │  Pod/Node │────▶│  Prometheus  │ (采集指标)       │
│  └──────────┘      └──────┬───────┘                 │
│                           │                          │
│                           ▼                          │
│                    ┌──────────────┐                  │
│                    │   Grafana    │ (可视化展示)      │
│                    └──────┬───────┘                  │
│                           │                          │
│                           ▼                          │
│                    ┌──────────────┐                  │
│                    │  告警通知    │ (钉钉/邮件等)     │
│                    └──────────────┘                  │
└─────────────────────────────────────────────────────┘
```

**典型监控栈**：
- **Prometheus**：采集和存储指标数据
- **Grafana**：可视化展示
- **Alertmanager**：告警处理
- **Loki**：日志收集
- **Node Exporter**：节点指标采集
- **kube-state-metrics**：K8s 资源状态指标

## 安装

### 在 K3s/K8s 中安装（推荐）

```bash
# 使用 Helm 安装
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 创建命名空间
kubectl create namespace monitoring

# 安装 Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set adminPassword=admin123 \
  --set service.type=LoadBalancer

# 获取 admin 密码
kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d

# 查看服务地址
kubectl get svc grafana -n monitoring
```

### 使用 kube-prometheus-stack（一体化方案）

```bash
# 安装 Prometheus + Grafana + Alertmanager 全套
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# 访问 Grafana
kubectl port-forward svc/kube-prometheus-grafana 3000:80 -n monitoring
# 浏览器访问 http://localhost:3000
# 默认账号：admin / prom-operator
```

### Docker 安装（单机测试）

```bash
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  -e GF_SECURITY_ADMIN_PASSWORD=admin123 \
  grafana/grafana
```

## 基本使用

### 1. 添加数据源

访问 `http://grafana-host:3000`，登录后进入 Configuration → Data Sources：

```yaml
# Prometheus 数据源配置示例
Name: Prometheus
Type: Prometheus
URL: http://prometheus-server.monitoring.svc.cluster.local:9090
Access: Server
```

### 2. 导入 Dashboard

方式一：使用 Dashboard ID
1. 访问 https://grafana.com/grafana/dashboards
2. 找到需要的 Dashboard（如 K8s 集群监控：315）
3. 在 Grafana 中 Import → 输入 ID → 选择数据源

方式二：导入 JSON 文件
1. 下载 Dashboard JSON
2. Import → Upload JSON file → 选择文件

### 3. 常用 Dashboard 推荐

| Dashboard ID | 名称 | 用途 |
|--------------|------|------|
| 315 | Prometheus Kubernetes Cluster | K8s 集群总览 |
| 3119 | Prometheus Kubernetes Nodes | 节点监控 |
| 8588 | Kubernetes / Compute Resources / Cluster | 集群资源使用 |
| 747 | Kubernetes / Compute Resources / Namespace | 命名空间资源 |
| 5217 | Kubernetes / Persistent Volumes | 持久卷监控 |
| 13332 | Loki Kubernetes Logs | 日志查询 |

## 常用查询示例

### Prometheus 查询（PromQL）

```promql
# CPU 使用率
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)

# 内存使用
sum(container_memory_working_set_bytes{namespace="default"}) by (pod)

# Pod 重启次数
increase(kube_pod_container_status_restarts_total{namespace="default"}[1h])

# 节点 CPU 使用率
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 磁盘使用率
(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100
```

## 告警配置

### 在 Grafana 中配置告警

1. **创建告警规则**
   - 在 Dashboard Panel 中点击 Alert → Create Alert
   - 设置条件和阈值

2. **配置通知渠道**
   - Notification channels → Add channel
   - 支持：Email、Slack、钉钉、企业微信、Webhook 等

### 钉钉告警示例

```yaml
Name: DingTalk
Type: DingDing
Url: https://oapi.dingtalk.com/robot/send?access_token=YOUR_TOKEN
Message Type: actionCard
```

### 企业微信告警示例

```yaml
Name: WeCom
Type: WeCom
Url: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
```

## 与 K3s 集成

### 在 K3s 中快速启用监控

```bash
# K3s 内置了 Prometheus Operator 支持
# 安装时启用监控
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -

# 安装 kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.service.type=LoadBalancer \
  --set grafana.service.port=80

# 访问 Grafana
kubectl get svc monitoring-grafana -n monitoring
```

## 常用面板配置

### K8s 集群监控面板

```json
{
  "panels": [
    {
      "title": "Cluster CPU Usage",
      "type": "gauge",
      "targets": [
        {
          "expr": "sum(rate(container_cpu_usage_seconds_total[5m])) / sum(machine_cpu_cores) * 100"
        }
      ]
    },
    {
      "title": "Cluster Memory Usage",
      "type": "gauge",
      "targets": [
        {
          "expr": "sum(container_memory_working_set_bytes) / sum(machine_memory_bytes) * 100"
        }
      ]
    },
    {
      "title": "Pod Count",
      "type": "stat",
      "targets": [
        {
          "expr": "count(kube_pod_info)"
        }
      ]
    }
  ]
}
```

## 常见问题

### 1. Grafana 无法连接 Prometheus

```bash
# 检查 Prometheus 服务
kubectl get svc -n monitoring | grep prometheus

# 检查 ServiceMonitor
kubectl get servicemonitor -n monitoring

# 查看 Prometheus Pod 日志
kubectl logs -l app=prometheus -n monitoring
```

### 2. Dashboard 无数据显示

- 确认数据源配置正确
- 检查时间范围是否合适
- 验证 PromQL 查询语法
- 确认 Prometheus 正在采集数据

### 3. 告警不触发

- 检查告警规则配置
- 确认通知渠道配置正确
- 查看 Grafana 日志排查问题

## 总结

Grafana 是 Kubernetes 监控体系中的核心组件，负责将 Prometheus 采集的指标数据可视化展示。配合 kube-prometheus-stack 可以快速搭建完整的监控告警体系，是 K3s/K8s 生产环境的必备工具。
