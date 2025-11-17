# 05. IAM and RBAC

## Table of Contents
1. [Overview](#overview)
2. [AWS IAM Strategy](#aws-iam-strategy)
3. [IRSA (IAM Roles for Service Accounts)](#irsa)
4. [Kubernetes RBAC](#kubernetes-rbac)
5. [Service Account Management](#service-account-management)
6. [Role Definitions](#role-definitions)
7. [Policy Examples](#policy-examples)
8. [Access Control Matrix](#access-control-matrix)
9. [Audit and Compliance](#audit-and-compliance)
10. [Best Practices](#best-practices)

---

## Overview

This document covers Identity and Access Management (IAM) strategies for both AWS resources and Kubernetes workloads, implementing least privilege access control.

### Access Control Layers

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: AWS IAM (Infrastructure Access)                   │
│  - User/Role-based access to AWS services                   │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: IRSA (Pod-to-AWS Access)                         │
│  - Kubernetes pods assume IAM roles via OIDC                │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Kubernetes RBAC (Cluster Access)                 │
│  - Control access to K8s API and resources                  │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: Application-Level (Business Logic)                │
│  - JWT tokens, OAuth, custom authorization                  │
└─────────────────────────────────────────────────────────────┘
```

---

## AWS IAM Strategy

### IAM User vs IAM Role

**Principle**: Never use IAM users with access keys for programmatic access. Always use IAM roles.

| Use Case | Method | Why |
|----------|--------|-----|
| **Human Access** | IAM User + MFA | Console access, temporary credentials |
| **EC2/EKS Nodes** | Instance Profile | Automatic credential rotation |
| **Pods in EKS** | IRSA (IAM Role) | No long-lived credentials |
| **CI/CD** | OIDC Provider | GitHub Actions without access keys |
| **Lambda** | Execution Role | Scoped permissions per function |

### IAM Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescriptiveStatementId",
      "Effect": "Allow",
      "Action": [
        "service:Action"
      ],
      "Resource": "arn:aws:service:region:account:resource",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

### Terraform IAM Policy Example

```hcl
# modules/iam/policies.tf

# S3 Read-Only Access Policy
resource "aws_iam_policy" "s3_read_only" {
  name        = "${var.environment}-s3-read-only"
  description = "Read-only access to specific S3 buckets"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ListBucket"
        Effect = "Allow"
        Action = [
          "s3:ListBucket",
          "s3:GetBucketLocation"
        ]
        Resource = "arn:aws:s3:::${var.bucket_name}"
      },
      {
        Sid    = "GetObjects"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:GetObjectVersion"
        ]
        Resource = "arn:aws:s3:::${var.bucket_name}/*"
      }
    ]
  })

  tags = {
    Environment = var.environment
  }
}

# RDS Access Policy
resource "aws_iam_policy" "rds_connect" {
  name        = "${var.environment}-rds-connect"
  description = "Allow RDS IAM authentication"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowRDSConnect"
        Effect = "Allow"
        Action = [
          "rds-db:connect"
        ]
        Resource = "arn:aws:rds-db:${var.region}:${data.aws_caller_identity.current.account_id}:dbuser:${aws_db_instance.main.resource_id}/*"
      }
    ]
  })
}

# Secrets Manager Access Policy
resource "aws_iam_policy" "secrets_read" {
  name        = "${var.environment}-secrets-read"
  description = "Read secrets from Secrets Manager"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadSecrets"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = "arn:aws:secretsmanager:${var.region}:${data.aws_caller_identity.current.account_id}:secret:${var.environment}/*"
      },
      {
        Sid    = "DecryptSecrets"
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey"
        ]
        Resource = aws_kms_key.secrets.arn
      }
    ]
  })
}
```

---

## IRSA (IAM Roles for Service Accounts)

### OIDC Provider Setup

```hcl
# modules/eks/irsa.tf

# Extract OIDC provider URL from EKS cluster
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

# Create OIDC provider
resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer

  tags = {
    Name = "${var.cluster_name}-oidc-provider"
  }
}

# Output for use in other modules
output "oidc_provider_arn" {
  value = aws_iam_openid_connect_provider.eks.arn
}

output "oidc_provider_url" {
  value = replace(aws_iam_openid_connect_provider.eks.url, "https://", "")
}
```

### IRSA Role for Application

```hcl
# modules/eks/irsa_roles.tf

# API Gateway S3 Access Role
resource "aws_iam_role" "api_gateway_s3" {
  name = "${var.cluster_name}-api-gateway-s3"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${var.oidc_provider_url}:sub" = "system:serviceaccount:production:api-gateway-sa"
            "${var.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = {
    Service = "api-gateway"
  }
}

resource "aws_iam_role_policy_attachment" "api_gateway_s3" {
  role       = aws_iam_role.api_gateway_s3.name
  policy_arn = aws_iam_policy.s3_read_only.arn
}

# External Secrets Operator Role
resource "aws_iam_role" "external_secrets" {
  name = "${var.cluster_name}-external-secrets"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${var.oidc_provider_url}:sub" = "system:serviceaccount:external-secrets:external-secrets-sa"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "external_secrets" {
  name = "secrets-manager-access"
  role = aws_iam_role.external_secrets.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret",
          "secretsmanager:ListSecrets"
        ]
        Resource = "arn:aws:secretsmanager:${var.region}:${data.aws_caller_identity.current.account_id}:secret:${var.environment}/*"
      }
    ]
  })
}

# AWS Load Balancer Controller Role
resource "aws_iam_role" "aws_load_balancer_controller" {
  name = "${var.cluster_name}-aws-load-balancer-controller"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${var.oidc_provider_url}:sub" = "system:serviceaccount:kube-system:aws-load-balancer-controller"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "aws_load_balancer_controller" {
  role       = aws_iam_role.aws_load_balancer_controller.name
  policy_arn = aws_iam_policy.aws_load_balancer_controller.arn
}
```

### Kubernetes ServiceAccount

```yaml
# k8s/base/api-gateway/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-gateway-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/eks-prod-api-gateway-s3
automountServiceAccountToken: true
```

### Using IRSA in Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  template:
    spec:
      serviceAccountName: api-gateway-sa
      containers:
      - name: api-gateway
        image: api-gateway:v1.0.0
        env:
        - name: AWS_REGION
          value: us-east-1
        # AWS SDK automatically uses IRSA credentials
```

---

## Kubernetes RBAC

### RBAC Components

| Resource | Scope | Purpose |
|----------|-------|---------|
| **Role** | Namespace | Permissions within a namespace |
| **ClusterRole** | Cluster-wide | Permissions across all namespaces |
| **RoleBinding** | Namespace | Bind Role to subjects in namespace |
| **ClusterRoleBinding** | Cluster-wide | Bind ClusterRole to subjects |

### Developer Role (Namespace-Scoped)

```yaml
# k8s/rbac/developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
# Pods
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "delete"]
# Deployments
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# ConfigMaps (read-only)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
# Secrets (read-only)
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
# HPA
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
- kind: User
  name: developer@example.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### Read-Only Role

```yaml
# k8s/rbac/viewer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: viewer
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "configmaps", "events"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: viewer-binding
  namespace: production
subjects:
- kind: Group
  name: viewers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: viewer
  apiGroup: rbac.authorization.k8s.io
```

### Cluster Admin Role

```yaml
# k8s/rbac/cluster-admin-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
- kind: User
  name: admin@example.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: cluster-admins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin  # Built-in role
  apiGroup: rbac.authorization.k8s.io
```

### CI/CD Service Account

```yaml
# k8s/rbac/cicd-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github-actions-deployer
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployer
rules:
# Deployments
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# Services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# ConfigMaps and Secrets
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "create", "update", "patch"]
# Pods (for status checking)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
# Ingress
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: deployer-binding
subjects:
- kind: ServiceAccount
  name: github-actions-deployer
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

---

## Access Control Matrix

### Human Users

| Role | AWS Console | EKS API | Deployments | Secrets | Nodes | Billing |
|------|-------------|---------|-------------|---------|-------|---------|
| **DevOps Lead** | Full | Full | ✅ | ✅ | ✅ | ✅ |
| **Senior Engineer** | Read | Full | ✅ | ✅ | Read | Read |
| **Developer** | Read | Namespace | Dev namespace | Read | Read | ❌ |
| **Viewer** | Read | Read-only | Read | ❌ | Read | ❌ |
| **Auditor** | Read-only | Read-only | Read | ❌ | Read | Read |

### Service Accounts

| Service | AWS IAM | K8s RBAC | Permissions |
|---------|---------|----------|-------------|
| **API Gateway** | IRSA | Namespace | S3 read, Secrets read |
| **External Secrets** | IRSA | Namespace | Secrets Manager full |
| **ALB Controller** | IRSA | Cluster | ALB/Target Group management |
| **Cluster Autoscaler** | IRSA | Cluster | ASG management |
| **Prometheus** | None | Cluster | Read pods/services/nodes |
| **GitHub Actions** | OIDC | Cluster | Deploy to all namespaces |

---

## Audit and Compliance

### Enable CloudTrail for IAM

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "${var.environment}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }

  tags = {
    Name = "${var.environment}-cloudtrail"
  }
}
```

### Kubernetes Audit Logging

```hcl
resource "aws_eks_cluster" "main" {
  # ... other config ...

  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]
}
```

### Review IAM Access

```bash
# List all IAM users
aws iam list-users

# Check user's attached policies
aws iam list-attached-user-policies --user-name USERNAME

# Check user's access keys
aws iam list-access-keys --user-name USERNAME

# Check last activity
aws iam get-user --user-name USERNAME

# Generate credential report
aws iam generate-credential-report
aws iam get-credential-report --output text | base64 -d > credential-report.csv
```

### Review Kubernetes RBAC

```bash
# List all service accounts
kubectl get sa -A

# Check permissions for service account
kubectl auth can-i --list --as=system:serviceaccount:production:api-gateway-sa

# Check who can perform action
kubectl auth can-i create pods --as=developer@example.com -n development

# Audit RBAC bindings
kubectl get rolebindings,clusterrolebindings -A -o wide
```

---

## Best Practices

### AWS IAM Best Practices

1. ✅ **Use IAM Roles, not Users**
   - Leverage IRSA for pods
   - Use OIDC for CI/CD
   - Instance profiles for EC2

2. ✅ **Enforce MFA**
   ```hcl
   resource "aws_iam_policy" "require_mfa" {
     policy = jsonencode({
       Statement = [{
         Effect = "Deny"
         Action = "*"
         Resource = "*"
         Condition = {
           BoolIfExists = {
             "aws:MultiFactorAuthPresent" = "false"
           }
         }
       }]
     })
   }
   ```

3. ✅ **Regular Access Reviews**
   - Quarterly audit of IAM users/roles
   - Remove unused access keys
   - Review CloudTrail logs

4. ✅ **Least Privilege**
   - Start with minimal permissions
   - Add permissions as needed
   - Use policy conditions

5. ✅ **Tag Everything**
   ```hcl
   tags = {
     Environment = var.environment
     ManagedBy   = "Terraform"
     Team        = "DevOps"
     CostCenter  = "Engineering"
   }
   ```

### Kubernetes RBAC Best Practices

1. ✅ **Namespace Isolation**
   - Separate namespaces per environment
   - Use namespace-scoped roles

2. ✅ **Service Account per Workload**
   - Don't use default service account
   - Create dedicated SA for each app

3. ✅ **Regular RBAC Audits**
   ```bash
   # Find overly permissive bindings
   kubectl get clusterrolebindings -o json | \
     jq '.items[] | select(.roleRef.name == "cluster-admin")'
   ```

4. ✅ **Use Groups**
   - Manage users via IAM/LDAP groups
   - Bind roles to groups, not individual users

5. ✅ **Document Permissions**
   - Maintain access control matrix
   - Document who has access to what

---

[← Back to Security Model](./04-security-model.md) | [Next: Application Architecture →](./06-application-architecture.md)
