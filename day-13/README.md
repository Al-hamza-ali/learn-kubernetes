# Day 13: Monitoring and Logging

## Overview

Observability is crucial for running production Kubernetes clusters. This includes **Monitoring** (metrics), **Logging** (events), and **Tracing** (request flows).

```
┌─────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY PILLARS                         │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   📊 METRICS         📝 LOGS           🔍 TRACES        │   │
│   │   (Prometheus)       (Loki/ELK)        (Jaeger)         │   │
│   │                                                          │   │
│   │   "What is          "What             "How did          │   │
│   │    happening?"       happened?"        requests flow?"  │   │
│   │                                                          │   │
│   │   CPU, Memory,      Application        Request path     │   │
│   │   Request rate,     errors,            across           │   │
│   │   Latency           Events             services         │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│                    📈 VISUALIZATION                              │
│                       (Grafana)                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Monitoring with Prometheus

### What is Prometheus?

**Prometheus** is an open-source monitoring system that:
- Collects metrics from configured targets
- Stores time-series data
- Provides powerful query language (PromQL)
- Triggers alerts based on conditions

### Prometheus Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS STACK                              │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Prometheus Server                     │   │
│   │                                                          │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │   │
│   │   │   Scraper   │  │    TSDB     │  │   Alert     │    │   │
│   │   │             │  │  (Storage)  │  │   Manager   │    │   │
│   │   └──────┬──────┘  └─────────────┘  └──────┬──────┘    │   │
│   └──────────┼───────────────────────────────────┼──────────┘   │
│              │                                   │               │
│              │ scrapes                           │ alerts        │
│              ▼                                   ▼               │
│   ┌─────────────────────┐                ┌─────────────────┐    │
│   │      Targets        │                │  Alertmanager   │    │
│   │                     │                │                 │    │
│   │  ┌───┐ ┌───┐ ┌───┐ │                │  Email, Slack,  │    │
│   │  │Pod│ │Pod│ │Pod│ │                │  PagerDuty      │    │
│   │  └───┘ └───┘ └───┘ │                └─────────────────┘    │
│   │                     │                                       │
│   │  ┌──────────────┐   │                                       │
│   │  │Node Exporter │   │                                       │
│   │  └──────────────┘   │                                       │
│   └─────────────────────┘                                       │
│                                                                  │
│              ┌─────────────────────────────────┐                │
│              │           Grafana               │                │
│              │    (Visualization Dashboard)    │                │
│              └─────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

### Installing Prometheus Stack

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (includes Grafana, Alertmanager)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123
```

### Access Grafana

```bash
# Port forward to Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Default credentials: admin / prom-operator (or your set password)
```

---

## Key Metrics to Monitor

### Node Metrics

| Metric | Description | PromQL |
|--------|-------------|--------|
| CPU Usage | CPU utilization | `node_cpu_seconds_total` |
| Memory Usage | Memory utilization | `node_memory_MemAvailable_bytes` |
| Disk Usage | Disk space | `node_filesystem_avail_bytes` |
| Network | Network I/O | `node_network_receive_bytes_total` |

### Pod Metrics

| Metric | Description | PromQL |
|--------|-------------|--------|
| CPU | Pod CPU usage | `container_cpu_usage_seconds_total` |
| Memory | Pod memory | `container_memory_working_set_bytes` |
| Restarts | Container restarts | `kube_pod_container_status_restarts_total` |

### Application Metrics

```yaml
# Pod with Prometheus annotations
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    ports:
    - containerPort: 8080
```

---

## PromQL Examples

```promql
# CPU usage percentage by pod
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod) * 100

# Memory usage by namespace
sum(container_memory_working_set_bytes{namespace!=""}) by (namespace)

# Request rate
sum(rate(http_requests_total[5m])) by (service)

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# 95th percentile latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Pod restarts in last hour
increase(kube_pod_container_status_restarts_total[1h])
```

---

## Alerting

### Alert Rules

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
spec:
  groups:
  - name: application
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m]))
        /
        sum(rate(http_requests_total[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: High error rate detected
        description: Error rate is {{ $value | humanizePercentage }}
    
    - alert: PodCrashLooping
      expr: |
        increase(kube_pod_container_status_restarts_total[1h]) > 5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: Pod {{ $labels.pod }} is crash looping
    
    - alert: HighMemoryUsage
      expr: |
        container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.9
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: Container {{ $labels.container }} memory usage > 90%
```

### Alertmanager Configuration

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
    
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'slack-notifications'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty'
    
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#alerts'
        send_resolved: true
    
    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: '<your-service-key>'
```

---

## Logging

### Logging Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      LOGGING STACK                               │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                     Application                          │   │
│   │                         │                                │   │
│   │                    stdout/stderr                         │   │
│   │                         ▼                                │   │
│   │              Container Runtime (containerd)              │   │
│   │                         │                                │   │
│   │              /var/log/containers/*.log                   │   │
│   │                         │                                │   │
│   └─────────────────────────┼───────────────────────────────┘   │
│                             │                                    │
│                             ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │               Log Collector (DaemonSet)                  │   │
│   │               (Fluentd / Fluent Bit / Promtail)         │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│                             ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  Log Storage                             │   │
│   │          (Elasticsearch / Loki / CloudWatch)            │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│                             ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  Visualization                           │   │
│   │              (Kibana / Grafana)                          │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Viewing Pod Logs

```bash
# View logs
kubectl logs my-pod
kubectl logs my-pod -c my-container    # Specific container
kubectl logs -f my-pod                 # Follow logs
kubectl logs --tail=100 my-pod         # Last 100 lines
kubectl logs --since=1h my-pod         # Last hour
kubectl logs -l app=myapp              # By label
kubectl logs -p my-pod                 # Previous container

# Logs from all pods in deployment
kubectl logs deployment/myapp
kubectl logs -f deployment/myapp --all-containers
```

### Installing Loki Stack

```bash
# Add Grafana repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki Stack
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=true \
  --set promtail.enabled=true
```

### Loki LogQL Examples

```logql
# All logs from a namespace
{namespace="production"}

# Logs from specific app
{app="myapp"}

# Filter by content
{app="myapp"} |= "error"
{app="myapp"} |~ "error|warning"
{app="myapp"} != "debug"

# JSON parsing
{app="myapp"} | json | level="error"

# Rate of errors
rate({app="myapp"} |= "error" [5m])

# Count errors by level
sum by (level) (count_over_time({app="myapp"} | json [1h]))
```

---

## ServiceMonitor (Prometheus Operator)

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  namespaceSelector:
    matchNames:
    - default
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

---

## Kubernetes Dashboard

```bash
# Install Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Create admin user
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

# Get token
kubectl -n kubernetes-dashboard create token admin-user

# Access dashboard
kubectl proxy
# Open: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

---

## Best Practices

### 1. Use Structured Logging
```go
// Good: Structured JSON
log.Info("request processed", 
    "user_id", userID, 
    "duration_ms", duration)

// Output: {"level":"info","msg":"request processed","user_id":"123","duration_ms":45}
```

### 2. Include Request IDs
```yaml
env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
```

### 3. Set Log Levels
```yaml
env:
- name: LOG_LEVEL
  value: "info"    # debug in dev, info/warn in prod
```

### 4. Monitor the Four Golden Signals
| Signal | What to Measure |
|--------|-----------------|
| **Latency** | Request duration |
| **Traffic** | Requests per second |
| **Errors** | Error rate |
| **Saturation** | Resource utilization |

### 5. Create Dashboards for Key Metrics
- Node health
- Pod status
- Application performance
- Error rates

---

## Resource Metrics Commands

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods
kubectl top pods -n kube-system
kubectl top pods --containers

# Sort by CPU/Memory
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

---

## Summary

| Component | Purpose | Tools |
|-----------|---------|-------|
| **Metrics** | Numerical measurements over time | Prometheus, Grafana |
| **Logs** | Event records | Loki, ELK, Fluentd |
| **Alerts** | Notifications on conditions | Alertmanager |
| **Dashboards** | Visual representation | Grafana |

### Key Takeaways

1. **Install kube-prometheus-stack** for complete monitoring
2. **Use structured logging** for easy parsing
3. **Create alerts** for critical conditions
4. **Monitor the four golden signals**
5. **Set up dashboards** for visibility
6. **Centralize logs** for easier debugging

---

## Next Steps

When things go wrong, you need to debug! Complete your learning with **[Day 14: Troubleshooting Guide](../day-14/README.md)**!
