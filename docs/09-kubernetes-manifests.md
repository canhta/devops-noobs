# 09. Kubernetes Manifests and Deployment Patterns

## Table of Contents
1. [Overview](#overview)
2. [Deployment Strategies](#deployment-strategies)
3. [Kustomize vs Helm](#kustomize-vs-helm)
4. [Base Manifests](#base-manifests)
5. [Environment Overlays](#environment-overlays)
6. [ConfigMaps and Secrets](#configmaps-and-secrets)
7. [Resource Management](#resource-management)
8. [Health Checks](#health-checks)
9. [Auto-scaling](#auto-scaling)
10. [Complete Examples](#complete-examples)

---

## Overview

This document provides comprehensive Kubernetes manifest examples and deployment patterns for the DevOps platform, covering both Kustomize and Helm approaches.

### Manifest Organization Strategy

```
k8s/
├── base/                          # Base configurations (environment-agnostic)
│   ├── api-gateway/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   └── kustomization.yaml
│   ├── auth-service/
│   ├── order-service/
│   └── common/
│       ├── namespace.yaml
│       ├── configmap.yaml
│       └── network-policy.yaml
└── overlays/
    ├── dev/                       # Development overrides
    │   ├── kustomization.yaml
    │   ├── replicas-patch.yaml
    │   └── resource-patch.yaml
    └── prod/                      # Production overrides
        ├── kustomization.yaml
        ├── replicas-patch.yaml
        └── resource-patch.yaml
```

### Deployment Workflow

```
Code Change → Docker Build → ECR Push → Update Manifests → Deploy to K8s

1. Developer commits code
2. CI builds Docker image
3. Image pushed to ECR with tag
4. Kustomize updates image tag
5. kubectl apply manifests
6. Rolling update begins
7. Health checks verify pods
8. Old pods terminated
```

---

## Base Manifests

### Deployment Configuration

```yaml
# k8s/base/api-gateway/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    app: api-gateway
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api-gateway
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: api-gateway-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: api-gateway
        image: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/api-gateway:latest
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 3000
          protocol: TCP
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        - name: KAFKA_BROKERS
          valueFrom:
            configMapKeyRef:
              name: kafka-config
              key: brokers
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: password
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - api-gateway
              topologyKey: kubernetes.io/hostname
```

### Service Configuration

```yaml
# k8s/base/api-gateway/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  labels:
    app: api-gateway
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
  selector:
    app: api-gateway
```

### HPA Configuration

```yaml
# k8s/base/api-gateway/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 15
      selectPolicy: Max
```

---

## Environment Overlays

### Kustomization Base

```yaml
# k8s/base/api-gateway/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: default

resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - hpa.yaml
  - serviceaccount.yaml

commonLabels:
  app.kubernetes.io/name: api-gateway
  app.kubernetes.io/component: backend
  app.kubernetes.io/managed-by: kustomize
```

### Development Overlay

```yaml
# k8s/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

resources:
  - ../../base/api-gateway
  - ../../base/auth-service
  - ../../base/order-service

commonLabels:
  environment: dev

patchesStrategicMerge:
  - replicas-patch.yaml
  - resource-patch.yaml

images:
  - name: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/api-gateway
    newTag: dev-latest

configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=debug
      - METRICS_ENABLED=true
```

```yaml
# k8s/overlays/dev/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 2
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  minReplicas: 2
  maxReplicas: 5
```

### Production Overlay

```yaml
# k8s/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base/api-gateway
  - ../../base/auth-service
  - ../../base/order-service
  - pod-disruption-budget.yaml

commonLabels:
  environment: prod

patchesStrategicMerge:
  - replicas-patch.yaml
  - resource-patch.yaml

images:
  - name: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/api-gateway
    newTag: v1.2.3

configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=info
      - METRICS_ENABLED=true
```

```yaml
# k8s/overlays/prod/pod-disruption-budget.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api-gateway
```

---

## ConfigMaps and Secrets

### ConfigMap

```yaml
# k8s/base/common/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-config
data:
  brokers: "kafka-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092"
  topics: "orders.created,orders.updated,users.registered"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  METRICS_ENABLED: "true"
  TRACING_ENABLED: "true"
```

### External Secrets

```yaml
# k8s/base/common/external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: database-credentials
    creationPolicy: Owner
  dataFrom:
  - extract:
      key: prod/database/credentials
```

---

## Resource Management

### Resource Quotas

```yaml
# k8s/base/common/resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "50"
    requests.memory: "100Gi"
    limits.cpu: "100"
    limits.memory: "200Gi"
    pods: "100"
```

### Limit Range

```yaml
# k8s/base/common/limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production
spec:
  limits:
  - max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    type: Container
```

---

## Complete Examples

### Apply with Kustomize

```bash
# Preview changes
kubectl kustomize k8s/overlays/dev

# Apply to development
kubectl apply -k k8s/overlays/dev

# Apply to production
kubectl apply -k k8s/overlays/prod

# Delete resources
kubectl delete -k k8s/overlays/dev
```

### Helm Chart Alternative

```yaml
# helm/api-gateway/Chart.yaml
apiVersion: v2
name: api-gateway
description: API Gateway microservice
type: application
version: 1.0.0
appVersion: "1.0.0"
```

```yaml
# helm/api-gateway/values.yaml
replicaCount: 3

image:
  repository: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/api-gateway
  pullPolicy: Always
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

env:
  - name: NODE_ENV
    value: "production"
  - name: PORT
    value: "3000"
```

```bash
# Install with Helm
helm install api-gateway ./helm/api-gateway \
  --namespace production \
  --create-namespace \
  --values ./helm/api-gateway/values-prod.yaml

# Upgrade
helm upgrade api-gateway ./helm/api-gateway \
  --namespace production \
  --values ./helm/api-gateway/values-prod.yaml
```

---

[← Back to Terraform Structure](./08-terraform-structure.md) | [Next: CI/CD Pipeline →](./10-cicd-pipeline.md)
