# 14. Logging Strategy

## Table of Contents
1. [Overview](#overview)
2. [Logging Architecture](#logging-architecture)
3. [Structured Logging](#structured-logging)
4. [Log Levels](#log-levels)
5. [Log Aggregation](#log-aggregation)
6. [Log Parsing](#log-parsing)
7. [Retention and Compliance](#retention-and-compliance)
8. [Query Examples](#query-examples)
9. [Troubleshooting with Logs](#troubleshooting-with-logs)
10. [Best Practices](#best-practices)

---

## Overview

This document outlines the logging strategy for the DevOps platform, covering structured logging, aggregation with Loki, and best practices.

### Logging Goals

- **Observability**: Understand system behavior
- **Debugging**: Troubleshoot issues quickly
- **Audit**: Track user actions and changes
- **Compliance**: Meet regulatory requirements
- **Performance**: Minimal impact on applications

### Logging Stack

```
┌─────────────────────────────────────────────────────────┐
│  Applications (JSON logs to stdout)                     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Promtail (DaemonSet on each node)                      │
│  - Collects logs from /var/log/pods                     │
│  - Parses and labels                                    │
│  - Adds metadata (namespace, pod, container)            │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Loki (Central log aggregation)                         │
│  - Indexes metadata only                                │
│  - Stores logs efficiently                              │
│  - 7-day retention                                      │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Grafana (Query and visualization)                      │
│  - LogQL queries                                        │
│  - Live tail                                            │
│  - Dashboards                                           │
└─────────────────────────────────────────────────────────┘
```

---

## Logging Architecture

### Application Logging

```typescript
// logger.ts - Winston configuration
import winston from 'winston';

const logFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  winston.format.errors({ stack: true }),
  winston.format.json()
);

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: logFormat,
  defaultMeta: {
    service: process.env.SERVICE_NAME,
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION,
  },
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      ),
    }),
  ],
});

// Add request ID middleware
import { v4 as uuidv4 } from 'uuid';

export function requestIdMiddleware(req, res, next) {
  req.id = req.headers['x-request-id'] || uuidv4();
  res.setHeader('x-request-id', req.id);
  next();
}
```

### NestJS Logger Configuration

```typescript
// main.ts
import { Logger } from '@nestjs/common';
import { WinstonModule } from 'nest-winston';
import * as winston from 'winston';

const app = await NestFactory.create(AppModule, {
  logger: WinstonModule.createLogger({
    transports: [
      new winston.transports.Console({
        format: winston.format.combine(
          winston.format.timestamp(),
          winston.format.ms(),
          winston.format.json()
        ),
      }),
    ],
  }),
});
```

---

## Structured Logging

### Log Format

```json
{
  "timestamp": "2023-11-15T10:30:45.123Z",
  "level": "info",
  "message": "Order created successfully",
  "service": "order-service",
  "environment": "production",
  "version": "1.2.3",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "user_12345",
  "orderId": "order_67890",
  "duration": 145,
  "statusCode": 200,
  "method": "POST",
  "path": "/api/orders"
}
```

### Logging Examples

```typescript
// Info log
logger.info('User login successful', {
  userId: user.id,
  email: user.email,
  requestId: req.id,
});

// Warning log
logger.warn('Rate limit approaching', {
  userId: user.id,
  requestCount: 95,
  limit: 100,
  requestId: req.id,
});

// Error log with stack trace
logger.error('Database connection failed', {
  error: err.message,
  stack: err.stack,
  query: sanitizedQuery,
  requestId: req.id,
});

// Request logging middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    logger.info('HTTP Request', {
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: Date.now() - start,
      requestId: req.id,
      userAgent: req.get('user-agent'),
      ip: req.ip,
    });
  });
  
  next();
});
```

---

## Log Levels

### Level Hierarchy

| Level | Priority | Use Case | Examples |
|-------|----------|----------|----------|
| **ERROR** | 0 | Errors requiring immediate attention | Exceptions, failed operations |
| **WARN** | 1 | Warning conditions | Deprecated API usage, rate limits |
| **INFO** | 2 | General informational messages | User actions, state changes |
| **HTTP** | 3 | HTTP request/response | API calls, status codes |
| **DEBUG** | 4 | Detailed debugging information | Variable values, flow control |
| **TRACE** | 5 | Very detailed tracing | Function entry/exit |

### Environment-Specific Levels

```typescript
const logLevel = {
  development: 'debug',
  staging: 'info',
  production: 'warn',
}[process.env.NODE_ENV || 'development'];
```

---

## Log Aggregation

### Promtail Configuration

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      # Parse JSON logs
      - json:
          expressions:
            timestamp: timestamp
            level: level
            message: message
            service: service
            requestId: requestId
      
      # Add labels
      - labels:
          level:
          service:
      
      # Extract timestamp
      - timestamp:
          source: timestamp
          format: RFC3339
      
      # Output format
      - output:
          source: message
    
    relabel_configs:
      # Add namespace label
      - source_labels:
          - __meta_kubernetes_namespace
        target_label: namespace
      
      # Add pod label
      - source_labels:
          - __meta_kubernetes_pod_name
        target_label: pod
      
      # Add container label
      - source_labels:
          - __meta_kubernetes_pod_container_name
        target_label: container
      
      # Drop system namespaces
      - action: drop
        regex: kube-.*
        source_labels:
          - __meta_kubernetes_namespace
```

---

## Log Parsing

### LogQL Queries

```logql
# Basic query
{namespace="production", app="api-gateway"}

# Filter by log level
{namespace="production"} |= "level=error"

# JSON parsing
{namespace="production"} | json | level="error"

# Regular expression
{app="api-gateway"} |~ "duration: [0-9]+ms"

# Line format
{app="api-gateway"} | json | line_format "{{.timestamp}} [{{.level}}] {{.message}}"

# Label format (extract duration as label)
{app="api-gateway"} | json | label_format duration="{{.duration}}ms"

# Filter by numeric value
{app="api-gateway"} | json | duration > 1000

# Rate queries
rate({namespace="production", level="error"}[5m])

# Count over time
count_over_time({namespace="production", level="error"}[1h])

# Sum by label
sum by (service) (rate({namespace="production"}[5m]))
```

### Common Query Patterns

```logql
# Find slow requests
{namespace="production"} 
  | json 
  | duration > 1000 
  | line_format "{{.path}} took {{.duration}}ms"

# Find errors with stack traces
{namespace="production", level="error"} 
  | json 
  | stack != "" 
  | line_format "{{.message}}\n{{.stack}}"

# Count errors by service
sum by (service) (
  count_over_time({namespace="production", level="error"}[1h])
)

# Top 10 slowest endpoints
topk(10,
  sum by (path) (
    avg_over_time({namespace="production"} | json | __error__="" | unwrap duration [5m])
  )
)
```

---

## Retention and Compliance

### Retention Policy

```yaml
# loki-config.yaml
limits_config:
  retention_period: 168h  # 7 days

table_manager:
  retention_deletes_enabled: true
  retention_period: 168h

chunk_store_config:
  max_look_back_period: 0s
```

### Archive to S3

```hcl
# Long-term storage for compliance
resource "aws_s3_bucket" "logs_archive" {
  bucket = "${var.environment}-logs-archive"
  
  lifecycle_rule {
    enabled = true
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 365  # 1 year retention
    }
  }
}
```

---

## Query Examples

### Grafana Explore

```logql
# Live tail (like tail -f)
{namespace="production", app="api-gateway"}

# Search for specific user activity
{namespace="production"} | json | userId="user_12345"

# Trace request across services
{namespace="production"} | json | requestId="550e8400-e29b-41d4-a716"

# Find authentication failures
{namespace="production", app="auth-service"} |= "authentication failed"

# Database query errors
{namespace="production"} | json | message =~ "database.*error"
```

### Alert Queries

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: loki-alerts
spec:
  groups:
  - name: logs
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate({namespace="production", level="error"}[5m])) > 10
      for: 5m
      annotations:
        summary: High error rate in logs
        description: More than 10 errors per second
```

---

## Troubleshooting with Logs

### Debug Workflow

```bash
# 1. Identify time range of issue
# 2. Filter by namespace/app
# 3. Look for errors
{namespace="production", level="error"} | json

# 4. Find correlation
{namespace="production"} | json | requestId="<id-from-step-3>"

# 5. Check related services
{namespace="production"} | json | userId="<user-from-step-4>"
```

### Common Patterns

```logql
# Find pod restarts
{namespace="production"} |= "Starting application"

# Database connection issues
{namespace="production"} |~ "connection.*refused|timeout|pool"

# Memory issues
{namespace="production"} |= "OutOfMemoryError"

# Slow Kafka consumers
{namespace="production", app=~".*consumer"} | json | lag > 1000
```

---

## Best Practices

### Do's ✅

1. **Use Structured Logging (JSON)**
   ```typescript
   logger.info('Order created', { orderId, userId });
   ```

2. **Include Context**
   - Request ID
   - User ID
   - Correlation ID
   - Timestamp

3. **Log Levels Appropriately**
   - ERROR: Exceptions only
   - WARN: Recoverable issues
   - INFO: Business events
   - DEBUG: Development only

4. **Add Sampling for High Volume**
   ```typescript
   if (Math.random() < 0.1) {  // 10% sample
     logger.debug('Detailed trace', data);
   }
   ```

5. **Sanitize Sensitive Data**
   ```typescript
   logger.info('User login', {
     email: maskEmail(user.email),
     // Never log passwords!
   });
   ```

### Don'ts ❌

1. ❌ Don't log sensitive data (passwords, tokens, PII)
2. ❌ Don't use string concatenation
3. ❌ Don't log in tight loops
4. ❌ Don't ignore log levels
5. ❌ Don't forget to add context

### Performance Tips

```typescript
// Lazy evaluation
logger.debug(() => `Expensive operation: ${expensiveFunction()}`);

// Conditional logging
if (logger.isLevelEnabled('debug')) {
  logger.debug('Details', expensiveData);
}

// Async logging (non-blocking)
logger.info('Message', data, { async: true });
```

---

[← Back to Monitoring Dashboards](./13-monitoring-dashboards.md) | [Next: Autoscaling Strategy →](./15-autoscaling-strategy.md)
