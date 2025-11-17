# 12. Observability Stack

## Table of Contents
1. [Overview](#overview)
2. [The Three Pillars](#the-three-pillars)
3. [Prometheus Setup](#prometheus-setup)
4. [Loki Setup](#loki-setup)
5. [Grafana Configuration](#grafana-configuration)
6. [Metrics Collection](#metrics-collection)
7. [Log Aggregation](#log-aggregation)
8. [Alerting](#alerting)
9. [Dashboard Examples](#dashboard-examples)
10. [Best Practices](#best-practices)

---

## Overview

This document covers the complete observability stack for the DevOps platform, implementing the three pillars of observability: metrics, logs, and traces.

### Observability Stack Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Applications                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   API    │  │  Auth    │  │  Order   │  │   User   │       │
│  │ Gateway  │  │ Service  │  │ Service  │  │ Service  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │ /metrics    │ /metrics    │ /metrics    │ /metrics     │
│       │             │             │             │               │
└───────┼─────────────┼─────────────┼─────────────┼───────────────┘
        │             │             │             │
        ├─────────────┴─────────────┴─────────────┤
        │                                          │
        ▼                                          ▼
  ┌──────────┐                              ┌──────────┐
  │Prometheus│                              │  Loki    │
  │          │                              │          │
  │ - Scrape │                              │ - Logs   │
  │ - Store  │                              │ - Query  │
  │ - Alert  │                              │ - Index  │
  └────┬─────┘                              └────┬─────┘
       │                                         │
       └─────────────┬──────────────────────────┘
                     ▼
              ┌─────────────┐
              │   Grafana   │
              │             │
              │ - Dashboards│
              │ - Alerts    │
              │ - Explore   │
              └─────────────┘
                     │
                     ▼
              ┌─────────────┐
              │ Alert       │
              │ Manager     │
              │             │
              │ Slack/Email │
              └─────────────┘
```

### Components

| Component | Purpose | Data Retention |
|-----------|---------|----------------|
| **Prometheus** | Metrics collection and storage | 15 days |
| **Loki** | Log aggregation and querying | 7 days |
| **Grafana** | Visualization and dashboards | N/A (queries data sources) |
| **Promtail** | Log shipping to Loki | N/A (agent) |
| **AlertManager** | Alert routing and management | N/A |
| **Node Exporter** | Infrastructure metrics | N/A (exporter) |
| **kube-state-metrics** | Kubernetes metrics | N/A (exporter) |

---

## The Three Pillars

### 1. Metrics (Prometheus)
- **What**: Quantitative measurements over time
- **Use Cases**: CPU/memory usage, request rates, error rates, latency
- **Retention**: 15 days

### 2. Logs (Loki)
- **What**: Event records from applications and infrastructure
- **Use Cases**: Debugging, audit trails, error investigation
- **Retention**: 7 days

### 3. Traces (Optional - Tempo/Jaeger)
- **What**: Request flow across distributed services
- **Use Cases**: Performance bottlenecks, service dependencies
- **Retention**: 3 days

---

## Prometheus Setup

### Helm Installation

```bash
# Add Prometheus community repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (includes Prometheus, AlertManager, Grafana)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values prometheus-values.yaml
```

### Prometheus Configuration

```yaml
# prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: "50GB"
    
    resources:
      requests:
        memory: "2Gi"
        cpu: "1000m"
      limits:
        memory: "4Gi"
        cpu: "2000m"
    
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    
    # Service discovery
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    
    # Additional scrape configs
    additionalScrapeConfigs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

grafana:
  adminPassword: "CHANGE_ME"
  
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-kube-prometheus-prometheus:9090
        access: proxy
        isDefault: true
      - name: Loki
        type: loki
        url: http://loki:3100
        access: proxy
```

### ServiceMonitor for Applications

```yaml
# servicemonitor-api-gateway.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-gateway
  namespace: production
  labels:
    app: api-gateway
spec:
  selector:
    matchLabels:
      app: api-gateway
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

---

## Loki Setup

### Helm Installation

```bash
# Add Grafana repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --values loki-values.yaml
```

### Loki Configuration

```yaml
# loki-values.yaml
loki:
  enabled: true
  
  persistence:
    enabled: true
    storageClassName: gp3
    size: 50Gi
  
  config:
    auth_enabled: false
    
    ingester:
      chunk_idle_period: 3m
      chunk_block_size: 262144
      chunk_retain_period: 1m
      max_transfer_retries: 0
      lifecycler:
        ring:
          kvstore:
            store: inmemory
          replication_factor: 1
    
    limits_config:
      enforce_metric_name: false
      reject_old_samples: true
      reject_old_samples_max_age: 168h
      retention_period: 168h  # 7 days
    
    schema_config:
      configs:
      - from: 2020-10-24
        store: boltdb-shipper
        object_store: filesystem
        schema: v11
        index:
          prefix: index_
          period: 24h
    
    server:
      http_listen_port: 3100
    
    storage_config:
      boltdb_shipper:
        active_index_directory: /data/loki/boltdb-shipper-active
        cache_location: /data/loki/boltdb-shipper-cache
        cache_ttl: 24h
        shared_store: filesystem
      filesystem:
        directory: /data/loki/chunks
    
    chunk_store_config:
      max_look_back_period: 0s
    
    table_manager:
      retention_deletes_enabled: true
      retention_period: 168h

promtail:
  enabled: true
  
  config:
    clients:
    - url: http://loki:3100/loki/api/v1/push
    
    scrapeConfigs:
    - job_name: kubernetes-pods
      pipeline_stages:
      - cri: {}
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: __host__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        replacement: $1
        separator: /
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_pod_name
        target_label: job
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_container_name
        target_label: container
      - replacement: /var/log/pods/*$1/*.log
        separator: /
        source_labels:
        - __meta_kubernetes_pod_uid
        - __meta_kubernetes_pod_container_name
        target_label: __path__
```

---

## Grafana Configuration

### Dashboard Provisioning

```yaml
# grafana-dashboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  kubernetes-cluster.json: |
    {
      "dashboard": {
        "title": "Kubernetes Cluster Monitoring",
        "panels": [
          {
            "title": "CPU Usage",
            "targets": [
              {
                "expr": "sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)",
                "legendFormat": "{{pod}}"
              }
            ]
          }
        ]
      }
    }
```

### Alert Rules

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: application-alerts
  namespace: monitoring
spec:
  groups:
  - name: application
    interval: 30s
    rules:
    - alert: HighErrorRate
      expr: |
        (
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
        ) > 0.05
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High error rate detected"
        description: "Service {{ $labels.service }} has error rate above 5% (current: {{ $value | humanizePercentage }})"
    
    - alert: HighLatency
      expr: |
        histogram_quantile(0.99,
          sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
        ) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High latency detected"
        description: "Service {{ $labels.service }} p99 latency is above 1s (current: {{ $value }}s)"
    
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod is crash looping"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
    
    - alert: HighMemoryUsage
      expr: |
        (
          container_memory_working_set_bytes{pod!=""}
          /
          container_spec_memory_limit_bytes{pod!=""}
        ) > 0.9
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} memory usage is above 90%"
```

### AlertManager Configuration

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
      resolve_timeout: 5m
      slack_api_url: 'SLACK_WEBHOOK_URL'
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'slack-notifications'
      routes:
      - match:
          severity: critical
        receiver: 'slack-critical'
      - match:
          severity: warning
        receiver: 'slack-warnings'
    
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    
    - name: 'slack-critical'
      slack_configs:
      - channel: '#alerts-critical'
        title: '❗ Critical Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    
    - name: 'slack-warnings'
      slack_configs:
      - channel: '#alerts-warnings'
        title: '⚠️ Warning: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

---

## Best Practices

### 1. Metrics Collection
- Scrape interval: 30s (balance between granularity and overhead)
- Use labels wisely (high cardinality = performance issues)
- Implement recording rules for expensive queries
- Set appropriate retention periods

### 2. Log Management
- Use structured logging (JSON format)
- Add context (request ID, user ID, service name)
- Set appropriate log levels (debug/info/warn/error)
- Implement log sampling for high-volume logs

### 3. Alerting
- Alert on symptoms, not causes
- Use appropriate severity levels
- Include actionable descriptions
- Set sensible thresholds and durations
- Avoid alert fatigue

---

[← Back to Container Registry](./11-container-registry.md) | [Next: Monitoring Dashboards →](./13-monitoring-dashboards.md)
