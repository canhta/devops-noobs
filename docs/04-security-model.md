# 04. Security Model

## Table of Contents
1. [Overview](#overview)
2. [Security Principles](#security-principles)
3. [Network Security](#network-security)
4. [IAM Security](#iam-security)
5. [Data Security](#data-security)
6. [Application Security](#application-security)
7. [Kubernetes Security](#kubernetes-security)
8. [Secrets Management](#secrets-management)
9. [Compliance and Auditing](#compliance-and-auditing)
10. [Security Checklist](#security-checklist)

---

## Overview

This document outlines the comprehensive security model for the DevOps platform, implementing defense-in-depth strategies across network, application, data, and infrastructure layers.

### Security Objectives

- **Confidentiality**: Protect sensitive data from unauthorized access
- **Integrity**: Ensure data accuracy and prevent tampering
- **Availability**: Maintain service availability and resilience
- **Auditability**: Track all access and changes
- **Compliance**: Meet industry security standards

### Security Layers

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 7: Application Security                              │
│  - Input validation, OWASP Top 10 protection                │
├─────────────────────────────────────────────────────────────┤
│  Layer 6: Kubernetes Security                               │
│  - RBAC, Network Policies, Pod Security Standards           │
├─────────────────────────────────────────────────────────────┤
│  Layer 5: Secrets Management                                │
│  - AWS Secrets Manager, External Secrets Operator           │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: IAM and Authentication                            │
│  - IRSA, OIDC, Least Privilege                             │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Data Security                                     │
│  - Encryption at rest and in transit                        │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Network Security                                  │
│  - Security Groups, NACLs, Private Subnets                  │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Infrastructure Security                           │
│  - VPC Isolation, WAF, Shield                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Security Principles

### 1. Defense in Depth

Multiple layers of security controls to protect against threats at different levels.

### 2. Least Privilege

Grant minimum necessary permissions for each service and user.

### 3. Zero Trust

Never trust, always verify - authenticate and authorize every request.

### 4. Encryption Everywhere

Encrypt data at rest and in transit by default.

### 5. Immutable Infrastructure

No manual changes to running systems - deploy new versions instead.

---

## Network Security

### Security Group Strategy

**Layered Security Groups**:
```
Internet → WAF → ALB SG → EKS Node SG → Pod Network Policies
```

### AWS WAF Configuration

```hcl
# modules/security/waf.tf
resource "aws_wafv2_web_acl" "main" {
  name  = "${var.environment}-web-acl"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  # Rate limiting rule
  rule {
    name     = "rate-limit"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }

  # AWS Managed Rules - Core Rule Set
  rule {
    name     = "aws-managed-core"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesCommonRuleSet"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedCoreRule"
      sampled_requests_enabled   = true
    }
  }

  # Block known bad inputs
  rule {
    name     = "aws-managed-known-bad-inputs"
    priority = 3

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "KnownBadInputs"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "${var.environment}-waf"
    sampled_requests_enabled   = true
  }

  tags = {
    Name = "${var.environment}-waf"
  }
}

# Associate WAF with ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

---

## IAM Security

### IAM Roles for Service Accounts (IRSA)

IRSA allows pods to assume IAM roles without using long-lived credentials.

```hcl
# modules/eks/irsa.tf

# OIDC Provider for EKS
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer

  tags = {
    Name = "${var.cluster_name}-oidc"
  }
}

# Example: S3 Access Role for Application
resource "aws_iam_role" "app_s3_access" {
  name = "${var.cluster_name}-app-s3-access"

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
          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:default:app-service-account"
          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })

  tags = {
    Name = "${var.cluster_name}-app-s3-access"
  }
}

resource "aws_iam_role_policy" "app_s3_access" {
  name = "s3-access"
  role = aws_iam_role.app_s3_access.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ]
      Resource = "arn:aws:s3:::${var.app_bucket}/*"
    }]
  })
}
```

**Kubernetes ServiceAccount**:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/eks-prod-app-s3-access
```

### IAM Policy Best Practices

1. **Least Privilege**: Grant only necessary permissions
2. **Use Managed Policies**: Leverage AWS managed policies when possible
3. **Regular Audits**: Review IAM policies quarterly
4. **MFA Required**: Enforce MFA for human users
5. **No Root Access**: Disable root account access keys

---

## Data Security

### Encryption at Rest

**RDS/Aurora**:
```hcl
resource "aws_db_instance" "main" {
  # ... other configuration ...
  
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
}

resource "aws_kms_key" "rds" {
  description             = "${var.environment} RDS encryption key"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  tags = {
    Name = "${var.environment}-rds-key"
  }
}
```

**EBS Volumes** (used by pods):
```hcl
resource "aws_eks_node_group" "main" {
  # ... other configuration ...

  launch_template {
    id      = aws_launch_template.nodes.id
    version = "$Latest"
  }
}

resource "aws_launch_template" "nodes" {
  # ... other configuration ...

  block_device_mappings {
    device_name = "/dev/xvda"

    ebs {
      volume_size           = 100
      volume_type           = "gp3"
      encrypted             = true
      kms_key_id            = aws_kms_key.ebs.arn
      delete_on_termination = true
    }
  }
}
```

**S3 Buckets**:
```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}
```

### Encryption in Transit

**TLS Everywhere**:
- ALB terminates TLS (ACM certificate)
- Pod-to-pod: mTLS with service mesh (optional)
- Database connections: SSL/TLS required
- Kafka: TLS encryption enabled

```hcl
# ALB Listener with TLS
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }
}

# Redirect HTTP to HTTPS
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

---

## Application Security

### OWASP Top 10 Mitigations

| Risk | Mitigation | Implementation |
|------|-----------|----------------|
| **Broken Access Control** | RBAC, JWT validation | NestJS Guards, K8s RBAC |
| **Cryptographic Failures** | TLS, encrypted storage | AWS KMS, ACM |
| **Injection** | Input validation, parameterized queries | NestJS ValidationPipe, TypeORM |
| **Insecure Design** | Threat modeling, security reviews | Architecture reviews |
| **Security Misconfiguration** | IaC, automated scanning | Terraform, Trivy |
| **Vulnerable Components** | Dependency scanning | Snyk, npm audit |
| **Auth Failures** | MFA, session management | AWS Cognito, JWT |
| **Software/Data Integrity** | Signed images, checksum validation | Cosign, SHA verification |
| **Logging Failures** | Centralized logging, alerting | Loki, Prometheus |
| **SSRF** | Input validation, network policies | Application layer + K8s |

### NestJS Security Configuration

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import helmet from 'helmet';
import * as compression from 'compression';
import rateLimit from 'express-rate-limit';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Security headers
  app.use(helmet());

  // Rate limiting
  app.use(
    rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100, // limit each IP to 100 requests per windowMs
    })
  );

  // Compression
  app.use(compression());

  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // Strip non-whitelisted properties
      forbidNonWhitelisted: true, // Throw error on non-whitelisted properties
      transform: true, // Auto-transform to DTO instances
    })
  );

  // CORS
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
    credentials: true,
  });

  await app.listen(3000);
}
bootstrap();
```

---

## Kubernetes Security

### Pod Security Standards

```yaml
# Pod Security Standards (enforced at namespace level)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Network Policies

```yaml
# Default Deny All
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Allow specific ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-gateway-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-gateway
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: auth-service
    ports:
    - protocol: TCP
      port: 3000
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### RBAC Configuration

```yaml
# Service account for application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-gateway-sa
  namespace: production

---
# Role for application (minimal permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-gateway-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get"]

---
# Bind role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-gateway-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: api-gateway-sa
  namespace: production
roleRef:
  kind: Role
  name: api-gateway-role
  apiGroup: rbac.authorization.k8s.io
```

---

## Secrets Management

### AWS Secrets Manager + External Secrets Operator

```hcl
# Terraform: Create secrets in AWS Secrets Manager
resource "aws_secretsmanager_secret" "db_credentials" {
  name = "${var.environment}/database/credentials"
  
  tags = {
    Environment = var.environment
  }
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = "app_user"
    password = random_password.db_password.result
    host     = aws_db_instance.main.address
    port     = 5432
    database = "app_db"
  })
}
```

**External Secrets Operator**:
```yaml
# Install ESO
apiVersion: v1
kind: Namespace
metadata:
  name: external-secrets

---
# SecretStore (connects to AWS Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

---
# ExternalSecret (syncs from AWS to K8s)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: database-credentials
    creationPolicy: Owner
  data:
  - secretKey: DATABASE_URL
    remoteRef:
      key: prod/database/credentials
      property: DATABASE_URL
```

---

## Compliance and Auditing

### AWS CloudTrail

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "${var.environment}-cloudtrail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]
    }
  }

  tags = {
    Name = "${var.environment}-cloudtrail"
  }
}
```

### AWS Config

```hcl
resource "aws_config_configuration_recorder" "main" {
  name     = "${var.environment}-config-recorder"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported = true
  }
}

resource "aws_config_delivery_channel" "main" {
  name           = "${var.environment}-config-delivery"
  s3_bucket_name = aws_s3_bucket.config.id

  depends_on = [aws_config_configuration_recorder.main]
}
```

---

## Security Checklist

### Infrastructure Security
- [ ] VPC Flow Logs enabled
- [ ] Security Groups follow least privilege
- [ ] NACLs configured for subnet protection
- [ ] WAF rules active on ALB
- [ ] S3 buckets have block public access enabled
- [ ] CloudTrail logging to dedicated bucket
- [ ] AWS Config enabled for compliance

### Encryption
- [ ] RDS/Aurora encryption at rest with KMS
- [ ] EBS volumes encrypted with KMS
- [ ] S3 buckets encrypted with KMS
- [ ] Secrets Manager for sensitive data
- [ ] TLS 1.2+ for all traffic
- [ ] Certificate auto-renewal configured

### Access Control
- [ ] IAM roles use IRSA (no static credentials in pods)
- [ ] MFA enforced for AWS Console access
- [ ] Root account access keys deleted
- [ ] IAM password policy configured
- [ ] Kubernetes RBAC configured
- [ ] Pod Security Standards enforced

### Application Security
- [ ] Container images scanned (Trivy/Snyk)
- [ ] Dependencies regularly updated
- [ ] Input validation on all endpoints
- [ ] Rate limiting configured
- [ ] CORS properly configured
- [ ] Security headers enabled (Helmet.js)

### Monitoring & Incident Response
- [ ] CloudWatch alarms for security events
- [ ] Log aggregation to Loki
- [ ] Incident response runbook documented
- [ ] Backup and restore procedures tested
- [ ] DR plan validated

---

[← Back to Network Architecture](./03-network-architecture.md) | [Next: IAM and RBAC →](./05-iam-rbac.md)
