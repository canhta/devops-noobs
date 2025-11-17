# 13. Monitoring Dashboards

## Table of Contents
1. [Overview](#overview)
2. [Grafana Dashboard Structure](#grafana-dashboard-structure)
3. [Kubernetes Cluster Dashboard](#kubernetes-cluster-dashboard)
4. [Application Performance Dashboard](#application-performance-dashboard)
5. [Database Dashboard](#database-dashboard)
6. [Kafka Dashboard](#kafka-dashboard)
7. [Alert Dashboard](#alert-dashboard)
8. [Dashboard Provisioning](#dashboard-provisioning)
9. [Custom Panels](#custom-panels)
10. [Best Practices](#best-practices)

---

## Overview

This document provides complete Grafana dashboard configurations for monitoring the DevOps platform, including JSON definitions and PromQL queries.

### Dashboard Categories

```
┌─────────────────────────────────────────────────────────┐
│  Infrastructure Dashboards                              │
│  - Kubernetes Cluster Overview                          │
│  - Node Metrics                                         │
│  - Network Traffic                                      │
├─────────────────────────────────────────────────────────┤
│  Application Dashboards                                 │
│  - API Gateway Performance                              │
│  - Service Health                                       │
│  - Request Metrics                                      │
├─────────────────────────────────────────────────────────┤
│  Data Layer Dashboards                                  │
│  - Database Performance                                 │
│  - Kafka Metrics                                        │
│  - Cache Performance                                    │
├─────────────────────────────────────────────────────────┤
│  Operational Dashboards                                 │
│  - Alert Overview                                       │
│  - Cost Tracking                                        │
│  - Deployment History                                   │
└─────────────────────────────────────────────────────────┘
```

---

## Grafana Dashboard Structure

### Dashboard JSON Template

```json
{
  "dashboard": {
    "title": "Dashboard Title",
    "uid": "unique-id",
    "timezone": "browser",
    "schemaVersion": 16,
    "refresh": "30s",
    "time": {
      "from": "now-6h",
      "to": "now"
    },
    "panels": [],
    "templating": {
      "list": []
    },
    "annotations": {
      "list": []
    }
  }
}
```

### Variables/Templates

```json
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_namespace_labels, namespace)",
        "current": {
          "selected": true,
          "text": "production",
          "value": "production"
        },
        "multi": false,
        "includeAll": false,
        "refresh": 1
      },
      {
        "name": "pod",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_pod_info{namespace=\"$namespace\"}, pod)",
        "current": {},
        "multi": true,
        "includeAll": true,
        "refresh": 1
      }
    ]
  }
}
```

---

## Kubernetes Cluster Dashboard

### Overview Stats Row

```json
{
  "panels": [
    {
      "id": 1,
      "title": "Cluster CPU Usage",
      "type": "stat",
      "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
      "targets": [
        {
          "expr": "sum(rate(node_cpu_seconds_total{mode!=\"idle\"}[5m])) / sum(rate(node_cpu_seconds_total[5m])) * 100",
          "refId": "A"
        }
      ],
      "options": {
        "graphMode": "area",
        "colorMode": "value",
        "orientation": "auto",
        "textMode": "auto"
      },
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 70, "color": "yellow"},
              {"value": 85, "color": "red"}
            ]
          }
        }
      }
    },
    {
      "id": 2,
      "title": "Cluster Memory Usage",
      "type": "stat",
      "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0},
      "targets": [
        {
          "expr": "(sum(node_memory_MemTotal_bytes) - sum(node_memory_MemAvailable_bytes)) / sum(node_memory_MemTotal_bytes) * 100",
          "refId": "A"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 80, "color": "yellow"},
              {"value": 90, "color": "red"}
            ]
          }
        }
      }
    },
    {
      "id": 3,
      "title": "Total Pods",
      "type": "stat",
      "gridPos": {"h": 4, "w": 6, "x": 12, "y": 0},
      "targets": [
        {
          "expr": "count(kube_pod_info)",
          "refId": "A"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short"
        }
      }
    },
    {
      "id": 4,
      "title": "Nodes Available",
      "type": "stat",
      "gridPos": {"h": 4, "w": 6, "x": 18, "y": 0},
      "targets": [
        {
          "expr": "count(kube_node_info)",
          "refId": "A"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"value": 0, "color": "red"},
              {"value": 2, "color": "yellow"},
              {"value": 3, "color": "green"}
            ]
          }
        }
      }
    }
  ]
}
```

### Node CPU/Memory Graph

```json
{
  "id": 5,
  "title": "Node CPU Usage",
  "type": "graph",
  "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4},
  "targets": [
    {
      "expr": "100 - (avg by (node) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
      "legendFormat": "{{node}}",
      "refId": "A"
    }
  ],
  "xaxis": {"mode": "time", "show": true},
  "yaxes": [
    {
      "format": "percent",
      "label": "CPU Usage",
      "show": true
    }
  ],
  "legend": {
    "show": true,
    "alignAsTable": true,
    "avg": true,
    "current": true,
    "max": true,
    "values": true
  }
}
```

### Pod Status Table

```json
{
  "id": 10,
  "title": "Pod Status by Namespace",
  "type": "table",
  "gridPos": {"h": 8, "w": 24, "x": 0, "y": 12},
  "targets": [
    {
      "expr": "sum by (namespace, phase) (kube_pod_status_phase)",
      "format": "table",
      "refId": "A"
    }
  ],
  "transformations": [
    {
      "id": "organize",
      "options": {
        "excludeByName": {"Time": true},
        "indexByName": {},
        "renameByName": {
          "namespace": "Namespace",
          "phase": "Status",
          "Value": "Count"
        }
      }
    }
  ]
}
```

---

## Application Performance Dashboard

### Request Rate, Latency, Error Rate (RED)

```json
{
  "panels": [
    {
      "id": 20,
      "title": "Request Rate",
      "type": "graph",
      "gridPos": {"h": 8, "w": 8, "x": 0, "y": 0},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{namespace=\"$namespace\", service=\"$service\"}[5m])) by (service)",
          "legendFormat": "{{service}}",
          "refId": "A"
        }
      ],
      "yaxes": [
        {
          "format": "reqps",
          "label": "Requests/sec"
        }
      ]
    },
    {
      "id": 21,
      "title": "Request Latency (p50, p95, p99)",
      "type": "graph",
      "gridPos": {"h": 8, "w": 8, "x": 8, "y": 0},
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{namespace=\"$namespace\"}[5m])) by (le, service))",
          "legendFormat": "p50 - {{service}}",
          "refId": "A"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace=\"$namespace\"}[5m])) by (le, service))",
          "legendFormat": "p95 - {{service}}",
          "refId": "B"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{namespace=\"$namespace\"}[5m])) by (le, service))",
          "legendFormat": "p99 - {{service}}",
          "refId": "C"
        }
      ],
      "yaxes": [
        {
          "format": "s",
          "label": "Latency"
        }
      ]
    },
    {
      "id": 22,
      "title": "Error Rate",
      "type": "graph",
      "gridPos": {"h": 8, "w": 8, "x": 16, "y": 0},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{namespace=\"$namespace\", status=~\"5..\"}[5m])) by (service) / sum(rate(http_requests_total{namespace=\"$namespace\"}[5m])) by (service) * 100",
          "legendFormat": "{{service}}",
          "refId": "A"
        }
      ],
      "yaxes": [
        {
          "format": "percent",
          "label": "Error %"
        }
      ],
      "alert": {
        "conditions": [
          {
            "evaluator": {
              "params": [5],
              "type": "gt"
            },
            "operator": {
              "type": "and"
            },
            "query": {
              "params": ["A", "5m", "now"]
            },
            "reducer": {
              "params": [],
              "type": "avg"
            },
            "type": "query"
          }
        ],
        "executionErrorState": "alerting",
        "frequency": "1m",
        "handler": 1,
        "name": "High Error Rate",
        "noDataState": "no_data"
      }
    }
  ]
}
```

### HTTP Status Codes

```json
{
  "id": 23,
  "title": "HTTP Status Codes",
  "type": "graph",
  "gridPos": {"h": 6, "w": 12, "x": 0, "y": 8},
  "targets": [
    {
      "expr": "sum by (status) (rate(http_requests_total{namespace=\"$namespace\", service=\"$service\"}[5m]))",
      "legendFormat": "{{status}}",
      "refId": "A"
    }
  ],
  "yaxes": [
    {
      "format": "reqps",
      "label": "Requests/sec"
    }
  ],
  "seriesOverrides": [
    {
      "alias": "/2../",
      "color": "green"
    },
    {
      "alias": "/4../",
      "color": "yellow"
    },
    {
      "alias": "/5../",
      "color": "red"
    }
  ]
}
```

---

## Database Dashboard

### RDS CloudWatch Metrics

```json
{
  "panels": [
    {
      "id": 30,
      "title": "Database Connections",
      "type": "graph",
      "datasource": "CloudWatch",
      "gridPos": {"h": 6, "w": 12, "x": 0, "y": 0},
      "targets": [
        {
          "namespace": "AWS/RDS",
          "metricName": "DatabaseConnections",
          "dimensions": {
            "DBInstanceIdentifier": ["prod-db"]
          },
          "statistic": "Average",
          "period": "300",
          "refId": "A"
        }
      ]
    },
    {
      "id": 31,
      "title": "Read/Write IOPS",
      "type": "graph",
      "datasource": "CloudWatch",
      "gridPos": {"h": 6, "w": 12, "x": 12, "y": 0},
      "targets": [
        {
          "namespace": "AWS/RDS",
          "metricName": "ReadIOPS",
          "dimensions": {
            "DBInstanceIdentifier": ["prod-db"]
          },
          "statistic": "Average",
          "period": "300",
          "refId": "A",
          "alias": "Read IOPS"
        },
        {
          "namespace": "AWS/RDS",
          "metricName": "WriteIOPS",
          "dimensions": {
            "DBInstanceIdentifier": ["prod-db"]
          },
          "statistic": "Average",
          "period": "300",
          "refId": "B",
          "alias": "Write IOPS"
        }
      ]
    },
    {
      "id": 32,
      "title": "CPU Utilization",
      "type": "gauge",
      "datasource": "CloudWatch",
      "gridPos": {"h": 6, "w": 6, "x": 0, "y": 6},
      "targets": [
        {
          "namespace": "AWS/RDS",
          "metricName": "CPUUtilization",
          "dimensions": {
            "DBInstanceIdentifier": ["prod-db"]
          },
          "statistic": "Average",
          "period": "300",
          "refId": "A"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 70, "color": "yellow"},
              {"value": 85, "color": "red"}
            ]
          }
        }
      }
    }
  ]
}
```

### Query Performance (Postgres)

```json
{
  "id": 35,
  "title": "Slow Queries",
  "type": "logs",
  "datasource": "Loki",
  "gridPos": {"h": 8, "w": 24, "x": 0, "y": 12},
  "targets": [
    {
      "expr": "{namespace=\"production\", app=\"postgres\"} |= \"duration\" | regexp \"duration: (?P<duration>\\\\d+\\\\.\\\\d+) ms\" | line_format \"{{.duration}}ms - {{.query}}\" | duration > 1000",
      "refId": "A"
    }
  ]
}
```

---

## Kafka Dashboard

### Kafka Metrics

```json
{
  "panels": [
    {
      "id": 40,
      "title": "Messages In Per Second",
      "type": "graph",
      "gridPos": {"h": 6, "w": 12, "x": 0, "y": 0},
      "targets": [
        {
          "expr": "sum(rate(kafka_server_brokertopicmetrics_messagesin_total[5m])) by (topic)",
          "legendFormat": "{{topic}}",
          "refId": "A"
        }
      ]
    },
    {
      "id": 41,
      "title": "Consumer Lag",
      "type": "graph",
      "gridPos": {"h": 6, "w": 12, "x": 12, "y": 0},
      "targets": [
        {
          "expr": "kafka_consumergroup_lag",
          "legendFormat": "{{consumergroup}} - {{topic}}",
          "refId": "A"
        }
      ],
      "alert": {
        "conditions": [
          {
            "evaluator": {
              "params": [10000],
              "type": "gt"
            },
            "query": {
              "params": ["A", "5m", "now"]
            },
            "reducer": {
              "type": "avg"
            },
            "type": "query"
          }
        ],
        "name": "High Consumer Lag"
      }
    },
    {
      "id": 42,
      "title": "Bytes In/Out",
      "type": "graph",
      "gridPos": {"h": 6, "w": 12, "x": 0, "y": 6},
      "targets": [
        {
          "expr": "sum(rate(kafka_server_brokertopicmetrics_bytesin_total[5m]))",
          "legendFormat": "Bytes In",
          "refId": "A"
        },
        {
          "expr": "sum(rate(kafka_server_brokertopicmetrics_bytesout_total[5m]))",
          "legendFormat": "Bytes Out",
          "refId": "B"
        }
      ],
      "yaxes": [
        {
          "format": "Bps"
        }
      ]
    }
  ]
}
```

---

## Dashboard Provisioning

### Grafana ConfigMap

```yaml
# grafana-dashboards-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  kubernetes-cluster.json: |
    {
      "dashboard": {
        "title": "Kubernetes Cluster",
        "uid": "k8s-cluster",
        "panels": [...]
      }
    }
  
  application-performance.json: |
    {
      "dashboard": {
        "title": "Application Performance",
        "uid": "app-perf",
        "panels": [...]
      }
    }
```

### Helm Values for Dashboard Provisioning

```yaml
# prometheus-values.yaml
grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default
  
  dashboards:
    default:
      kubernetes-cluster:
        url: https://grafana.com/api/dashboards/7249/revisions/1/download
      
      node-exporter:
        gnetId: 1860
        revision: 27
        datasource: Prometheus
```

---

## Custom Panels

### SLO Dashboard

```json
{
  "id": 50,
  "title": "SLO: Availability (99.9%)",
  "type": "stat",
  "gridPos": {"h": 4, "w": 8, "x": 0, "y": 0},
  "targets": [
    {
      "expr": "(sum(rate(http_requests_total{status!~\"5..\"}[30d])) / sum(rate(http_requests_total[30d]))) * 100",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "decimals": 3,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {"value": 0, "color": "red"},
          {"value": 99.9, "color": "green"}
        ]
      }
    }
  }
}
```

### Error Budget

```json
{
  "id": 51,
  "title": "Error Budget Remaining",
  "type": "gauge",
  "gridPos": {"h": 6, "w": 8, "x": 8, "y": 0},
  "targets": [
    {
      "expr": "((sum(rate(http_requests_total[30d])) * 0.001) - sum(rate(http_requests_total{status=~\"5..\"}[30d]))) / (sum(rate(http_requests_total[30d])) * 0.001) * 100",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "min": 0,
      "max": 100,
      "thresholds": {
        "steps": [
          {"value": 0, "color": "red"},
          {"value": 25, "color": "yellow"},
          {"value": 50, "color": "green"}
        ]
      }
    }
  }
}
```

---

## Best Practices

### Dashboard Design

1. **Start with Overview** - High-level metrics first
2. **Group Related Metrics** - Use rows to organize
3. **Use Variables** - Make dashboards reusable
4. **Consistent Color Scheme** - Use thresholds
5. **Add Annotations** - Mark deployments, incidents

### Performance

- Limit time range for expensive queries
- Use recording rules for complex calculations
- Avoid too many panels (< 20 per dashboard)
- Use appropriate refresh rates (30s-1m)

### Maintenance

```bash
# Export dashboard
curl -H "Authorization: Bearer $GRAFANA_API_KEY" \
  http://grafana:3000/api/dashboards/uid/k8s-cluster > dashboard.json

# Import dashboard via API
curl -X POST -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -d @dashboard.json \
  http://grafana:3000/api/dashboards/db
```

---

[← Back to Observability Stack](./12-observability-stack.md) | [Next: Logging Strategy →](./14-logging-strategy.md)
