# 15. Autoscaling Strategy

## Table of Contents
1. [Overview](#overview)
2. [Horizontal Pod Autoscaler](#horizontal-pod-autoscaler)
3. [Vertical Pod Autoscaler](#vertical-pod-autoscaler)
4. [Cluster Autoscaler](#cluster-autoscaler)
5. [Karpenter](#karpenter)
6. [Kafka Scaling](#kafka-scaling)
7. [Database Scaling](#database-scaling)
8. [Load Testing](#load-testing)
9. [Scaling Policies](#scaling-policies)
10. [Best Practices](#best-practices)

---

## Overview

This document covers comprehensive autoscaling strategies for the DevOps platform, from pod-level to cluster-level scaling, ensuring optimal resource utilization and cost efficiency.

### Autoscaling Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 4: Application Load Balancer                             │
│  - Target-based scaling                                         │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: Horizontal Pod Autoscaler (HPA)                      │
│  - CPU, Memory, Custom Metrics                                  │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Vertical Pod Autoscaler (VPA)                        │
│  - Right-size resource requests/limits                          │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: Cluster Autoscaler / Karpenter                       │
│  - Add/remove worker nodes                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Autoscaling Flow

```
Traffic Increase → HPA scales pods → Pods pending → Cluster Autoscaler 
adds nodes → Pods scheduled → Traffic handled → Traffic decreases → 
HPA scales down pods → Nodes underutilized → Cluster Autoscaler 
removes nodes
```

---

## Horizontal Pod Autoscaler

### CPU/Memory Based HPA

```yaml
# hpa-api-gateway.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
  namespace: production
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
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 50  # Max 50% of pods removed per period
        periodSeconds: 60
      - type: Pods
        value: 2  # Max 2 pods removed per period
        periodSeconds: 60
      selectPolicy: Min
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100  # Max 100% increase per period
        periodSeconds: 15
      - type: Pods
        value: 4  # Max 4 pods added per period
        periodSeconds: 15
      selectPolicy: Max
```

### Custom Metrics HPA

```yaml
# Install Prometheus Adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --values prometheus-adapter-values.yaml
```

```yaml
# prometheus-adapter-values.yaml
rules:
  default: false
  custom:
  - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
    resources:
      template: <<.Resource>>
    name:
      matches: "^(.*)_total"
      as: "${1}_per_second"
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
  
  - seriesQuery: 'http_request_duration_seconds_bucket{namespace!="",pod!=""}'
    resources:
      template: <<.Resource>>
    name:
      matches: "^(.*)_bucket"
      as: "${1}_p99"
    metricsQuery: 'histogram_quantile(0.99, sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (le, <<.GroupBy>>))'
```

```yaml
# HPA with custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-custom
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"  # Scale when > 1000 req/s per pod
  - type: Pods
    pods:
      metric:
        name: http_request_duration_seconds_p99
      target:
        type: AverageValue
        averageValue: "500m"  # Scale when p99 > 500ms
```

---

## Vertical Pod Autoscaler

### VPA Installation

```bash
# Install VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

### VPA Configuration

```yaml
# vpa-api-gateway.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-gateway
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  updatePolicy:
    updateMode: "Auto"  # Auto, Recreate, Initial, or Off
  resourcePolicy:
    containerPolicies:
    - containerName: api-gateway
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 4Gi
      controlledResources:
      - cpu
      - memory
      mode: Auto
```

### VPA Recommendations Only (No Auto-Apply)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-gateway-recommendations
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  updatePolicy:
    updateMode: "Off"  # Only provide recommendations
```

```bash
# View VPA recommendations
kubectl describe vpa api-gateway-recommendations -n production
```

---

## Cluster Autoscaler

### Installation

```bash
# Add autoscaler Helm repo
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update

# Install Cluster Autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=eks-prod \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT_ID:role/cluster-autoscaler
```

### Terraform Configuration

```hcl
# modules/eks/cluster-autoscaler.tf
resource "aws_iam_role" "cluster_autoscaler" {
  name = "${var.cluster_name}-cluster-autoscaler"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.eks.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:kube-system:cluster-autoscaler"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "cluster_autoscaler" {
  name = "cluster-autoscaler"
  role = aws_iam_role.cluster_autoscaler.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "autoscaling:DescribeAutoScalingGroups",
          "autoscaling:DescribeAutoScalingInstances",
          "autoscaling:DescribeLaunchConfigurations",
          "autoscaling:DescribeScalingActivities",
          "autoscaling:DescribeTags",
          "ec2:DescribeInstanceTypes",
          "ec2:DescribeLaunchTemplateVersions"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "autoscaling:SetDesiredCapacity",
          "autoscaling:TerminateInstanceInAutoScalingGroup",
          "ec2:DescribeImages",
          "ec2:GetInstanceTypesFromInstanceRequirements",
          "eks:DescribeNodegroup"
        ]
        Resource = "*"
      }
    ]
  })
}
```

### Node Group Tags (for Auto-Discovery)

```hcl
resource "aws_eks_node_group" "main" {
  # ... other configuration ...

  tags = {
    "k8s.io/cluster-autoscaler/${var.cluster_name}" = "owned"
    "k8s.io/cluster-autoscaler/enabled"             = "true"
  }
}
```

---

## Karpenter

### Karpenter vs Cluster Autoscaler

| Feature | Cluster Autoscaler | Karpenter |
|---------|-------------------|----------|
| **Speed** | Minutes | Seconds |
| **Node Selection** | Limited by ASG | Flexible |
| **Cost Optimization** | Basic | Advanced (spot, binpacking) |
| **Complexity** | Lower | Higher |
| **AWS Native** | No | Yes |

### Karpenter Installation

```bash
# Install Karpenter
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT_ID:role/karpenter-controller \
  --set settings.clusterName=eks-prod \
  --set settings.clusterEndpoint=$(aws eks describe-cluster --name eks-prod --query "cluster.endpoint" --output text) \
  --set settings.defaultInstanceProfile=KarpenterNodeInstanceProfile-eks-prod
```

### Provisioner Configuration

```yaml
# karpenter-provisioner.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values: ["spot", "on-demand"]
  - key: kubernetes.io/arch
    operator: In
    values: ["amd64"]
  - key: karpenter.k8s.aws/instance-category
    operator: In
    values: ["c", "m", "r"]
  - key: karpenter.k8s.aws/instance-generation
    operator: Gt
    values: ["4"]
  
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  
  providerRef:
    name: default
  
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 604800  # 7 days
  
  weight: 10
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: eks-prod
  securityGroupSelector:
    karpenter.sh/discovery: eks-prod
  instanceProfile: KarpenterNodeInstanceProfile-eks-prod
  amiFamily: AL2
  tags:
    Name: karpenter-node
    Environment: production
  userData: |
    #!/bin/bash
    /etc/eks/bootstrap.sh eks-prod
```

---

## Load Testing

### K6 Load Test

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp up to 200 users
    { duration: '5m', target: 200 },  // Stay at 200 users
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests must complete below 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
  },
};

export default function () {
  const res = http.get('https://api.example.com/orders');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

```bash
# Run load test
k6 run --vus 10 --duration 30s load-test.js

# Run with Cloud output
k6 run --out cloud load-test.js
```

### Monitor During Load Test

```bash
# Watch HPA scaling
kubectl get hpa -n production -w

# Watch pod count
kubectl get pods -n production -l app=api-gateway -w

# Watch node count
kubectl get nodes -w

# Check Cluster Autoscaler logs
kubectl logs -f -n kube-system -l app=cluster-autoscaler
```

---

## Best Practices

### HPA Best Practices
1. Set conservative scale-down policies to prevent flapping
2. Use multiple metrics for better scaling decisions
3. Set appropriate min/max replicas
4. Test scaling behavior under load
5. Monitor HPA metrics and events

### VPA Best Practices
1. Start with "Off" mode to review recommendations
2. Don't use VPA and HPA on same metric
3. Set reasonable min/max resource limits
4. Be aware of pod disruptions during updates
5. Use VPA for stateful workloads carefully

### Cluster Autoscaling Best Practices
1. Use Pod Disruption Budgets
2. Set appropriate node group min/max sizes
3. Use node affinity/taints for workload placement
4. Monitor scale-up/down events
5. Consider using mixed instance types

---

[← Back to Logging Strategy](./14-logging-strategy.md) | [Next: Performance Optimization →](./16-performance-optimization.md)
