# 18. Disaster Recovery

## Table of Contents
1. [Overview](#overview)
2. [RTO and RPO Objectives](#rto-and-rpo-objectives)
3. [Backup Strategy](#backup-strategy)
4. [Recovery Procedures](#recovery-procedures)
5. [Infrastructure Rebuild](#infrastructure-rebuild)
6. [Data Restoration](#data-restoration)
7. [Testing DR Plan](#testing-dr-plan)
8. [Multi-Region Considerations](#multi-region-considerations)
9. [Communication Plan](#communication-plan)
10. [DR Checklist](#dr-checklist)

---

## Overview

This document outlines the disaster recovery (DR) plan for the DevOps platform, including backup procedures, recovery objectives, and step-by-step restoration processes.

### Disaster Scenarios

| Scenario | Likelihood | Impact | RTO | RPO |
|----------|-----------|--------|-----|-----|
| **Single AZ Failure** | Medium | Low | < 5 min | 0 |
| **Region-wide Outage** | Low | High | < 4 hours | < 15 min |
| **Data Corruption** | Low | Medium | < 2 hours | < 1 hour |
| **Security Breach** | Low | High | < 8 hours | 0 |
| **Accidental Deletion** | Medium | Medium | < 1 hour | < 15 min |
| **Complete Infrastructure Loss** | Very Low | Critical | < 24 hours | < 1 hour |

### DR Architecture

```
Primary Region (us-east-1)              Backup Region (us-west-2)
┌─────────────────────────┐            ┌─────────────────────────┐
│  Production Environment │            │   DR Environment        │
│                         │            │   (Pilot Light)         │
│  ┌───────────────────┐ │            │  ┌───────────────────┐  │
│  │  EKS Cluster      │ │   Replicate│  │  EKS Cluster      │  │
│  │  - Active         │ ├───────────▶│  │  - Standby        │  │
│  └───────────────────┘ │            │  └───────────────────┘  │
│                         │            │                         │
│  ┌───────────────────┐ │   Replicate│  ┌───────────────────┐  │
│  │  Aurora Database  │ ├───────────▶│  │  Aurora Replica   │  │
│  │  - Multi-AZ       │ │            │  │  - Read Replica   │  │
│  └───────────────────┘ │            │  └───────────────────┘  │
│                         │            │                         │
│  ┌───────────────────┐ │   Replicate│  ┌───────────────────┐  │
│  │  S3 Backups       │ ├───────────▶│  │  S3 Backups       │  │
│  │  - Versioning     │ │  (CRR)     │  │  - Versioning     │  │
│  └───────────────────┘ │            │  └───────────────────┘  │
└─────────────────────────┘            └─────────────────────────┘
```

---

## RTO and RPO Objectives

### Definitions

- **RTO (Recovery Time Objective)**: Maximum acceptable downtime
- **RPO (Recovery Point Objective)**: Maximum acceptable data loss

### Target Objectives

| Component | RTO | RPO | Strategy |
|-----------|-----|-----|----------|
| **EKS Cluster** | 4 hours | 0 | Multi-AZ, Velero backups |
| **RDS Aurora** | 15 minutes | 5 minutes | Multi-AZ, automated backups, read replicas |
| **Kafka (MSK)** | 1 hour | 0 | Multi-AZ, replication |
| **Application State** | 30 minutes | 15 minutes | Stateless design, persistent volumes backed up |
| **Secrets** | 1 hour | 0 | AWS Secrets Manager replication |
| **Container Images** | 30 minutes | 0 | Multi-region ECR replication |

---

## Backup Strategy

### Kubernetes Resources (Velero)

```bash
# Install Velero with AWS plugin
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket eks-backup-prod \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero \
  --use-node-agent
```

**Backup Schedule**:
```yaml
# velero-schedules.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: hourly-backup
  namespace: velero
spec:
  schedule: "0 * * * *"  # Every hour
  template:
    includedNamespaces:
    - production
    ttl: 168h  # 7 days
---
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-full-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  template:
    includedNamespaces:
    - "*"
    ttl: 720h  # 30 days
```

### RDS Aurora Backup

```hcl
# Terraform configuration
resource "aws_rds_cluster" "main" {
  # ... other config ...
  
  backup_retention_period = 35  # 35 days
  preferred_backup_window = "03:00-04:00"  # UTC
  
  # Point-in-time restore
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  # Automated backups to S3
  s3_import {
    source_engine         = "postgres"
    source_engine_version = "14.9"
  }
}

# Manual snapshot
resource "aws_db_cluster_snapshot" "manual" {
  db_cluster_identifier          = aws_rds_cluster.main.id
  db_cluster_snapshot_identifier = "pre-upgrade-snapshot-${formatdate("YYYY-MM-DD", timestamp())}"
}
```

### ECR Multi-Region Replication

```hcl
# Primary region ECR
resource "aws_ecr_repository" "api_gateway" {
  name = "api-gateway"
  
  image_scanning_configuration {
    scan_on_push = true
  }
}

# Replication configuration
resource "aws_ecr_replication_configuration" "main" {
  replication_configuration {
    rule {
      destination {
        region      = "us-west-2"
        registry_id = data.aws_caller_identity.current.account_id
      }
      
      repository_filter {
        filter      = "*"
        filter_type = "PREFIX_MATCH"
      }
    }
  }
}
```

### Secrets Replication

```hcl
# Primary secret
resource "aws_secretsmanager_secret" "db_credentials" {
  name = "prod/database/credentials"
  
  replica {
    region = "us-west-2"
  }
}
```

---

## Recovery Procedures

### Complete Cluster Loss

**Scenario**: Entire EKS cluster is lost

```bash
#!/bin/bash
# disaster-recovery.sh

set -e

REGION="us-east-1"
CLUSTER_NAME="eks-prod"
BACKUP_NAME="daily-full-backup-20231115"

echo "Starting disaster recovery process..."

# Step 1: Recreate infrastructure with Terraform
echo "Step 1: Recreating infrastructure..."
cd terraform/environments/prod
terraform init
terraform apply -auto-approve

# Step 2: Configure kubectl
echo "Step 2: Configuring kubectl..."
aws eks update-kubeconfig --name $CLUSTER_NAME --region $REGION

# Step 3: Verify cluster access
echo "Step 3: Verifying cluster access..."
kubectl get nodes

# Step 4: Install core add-ons
echo "Step 4: Installing core add-ons..."
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.15/config/master/aws-k8s-cni.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.0/docs/install/v2_6_0_full.yaml

# Step 5: Install Velero
echo "Step 5: Installing Velero..."
velero install \
  --provider aws \
  --bucket eks-backup-prod \
  --backup-location-config region=$REGION \
  --snapshot-location-config region=$REGION \
  --secret-file ./credentials-velero

# Step 6: Restore from backup
echo "Step 6: Restoring from backup..."
velero restore create disaster-recovery-$(date +%Y%m%d%H%M) \
  --from-backup $BACKUP_NAME \
  --wait

# Step 7: Verify restoration
echo "Step 7: Verifying restoration..."
kubectl get pods -A
kubectl get svc -A

# Step 8: Run smoke tests
echo "Step 8: Running smoke tests..."
curl -f https://api.example.com/health || echo "Health check failed"

echo "Disaster recovery complete!"
```

### Database Recovery

#### Point-in-Time Recovery

```bash
# Restore to specific point in time
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier prod-aurora-cluster \
  --db-cluster-identifier prod-aurora-cluster-restored \
  --restore-to-time 2023-11-15T10:00:00Z \
  --use-latest-restorable-time

# Wait for restore to complete
aws rds wait db-cluster-available \
  --db-cluster-identifier prod-aurora-cluster-restored

# Update application connection string
kubectl create secret generic database-credentials \
  --from-literal=host=prod-aurora-cluster-restored.cluster-xxxxx.us-east-1.rds.amazonaws.com \
  --from-literal=password=<new-password> \
  -n production \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart pods to pick up new connection
kubectl rollout restart deployment -n production
```

#### Snapshot Recovery

```bash
# List available snapshots
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier prod-aurora-cluster

# Restore from snapshot
aws rds restore-db-cluster-from-snapshot \
  --db-cluster-identifier prod-aurora-cluster-restored \
  --snapshot-identifier manual-snapshot-20231115 \
  --engine aurora-postgresql
```

### Application Rollback

```bash
# Check deployment history
kubectl rollout history deployment/api-gateway -n production

# Rollback to previous version
kubectl rollout undo deployment/api-gateway -n production

# Rollback to specific revision
kubectl rollout undo deployment/api-gateway -n production --to-revision=5

# Verify rollback
kubectl rollout status deployment/api-gateway -n production
```

---

## Testing DR Plan

### Quarterly DR Test

```bash
#!/bin/bash
# dr-test.sh - Run quarterly DR test in isolated environment

TEST_CLUSTER="eks-dr-test"
TEST_NAMESPACE="dr-test"

echo "=== DR Test Started at $(date) ==="

# 1. Create test cluster
echo "Creating test cluster..."
eksctl create cluster \
  --name $TEST_CLUSTER \
  --region us-east-1 \
  --nodegroup-name test-nodes \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3

# 2. Install Velero
echo "Installing Velero..."
velero install --provider aws --bucket eks-backup-prod

# 3. Restore latest backup
echo "Restoring from backup..."
LATEST_BACKUP=$(velero backup get --output json | jq -r '.items[0].metadata.name')
velero restore create dr-test-$(date +%Y%m%d) --from-backup $LATEST_BACKUP

# 4. Wait for restore
echo "Waiting for restore to complete..."
velero restore describe dr-test-$(date +%Y%m%d) --details

# 5. Validate
echo "Validating restored resources..."
kubectl get all -n $TEST_NAMESPACE

# 6. Run tests
echo "Running smoke tests..."
# Add your test commands here

# 7. Cleanup
echo "Cleaning up test cluster..."
eksctl delete cluster --name $TEST_CLUSTER

echo "=== DR Test Completed at $(date) ==="
```

### DR Test Checklist

- [ ] Infrastructure can be recreated from Terraform
- [ ] Velero can restore all Kubernetes resources
- [ ] Database can be restored from snapshot
- [ ] Application comes up and passes health checks
- [ ] All secrets are available
- [ ] Monitoring and alerting works
- [ ] DNS and certificates are valid
- [ ] Document lessons learned
- [ ] Update DR runbook

---

## Multi-Region Considerations

### Active-Passive Setup

```
Primary Region (us-east-1)     Secondary Region (us-west-2)
┌───────────────────────   ┌───────────────────────
│ EKS Cluster (Active)  │   │ EKS Cluster (Standby) │
│ Aurora (Primary)      │   │ Aurora (Read Replica) │
│ MSK                   │   │ MSK (Mirror Maker)    │
│ S3 Backups            │   │ S3 Replicated         │
└───────────────────────   └───────────────────────
         │                            │
         └──────── Replication ────────┘
```

### Route53 Failover

```hcl
# Primary endpoint
resource "aws_route53_record" "api_primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "primary"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  health_check_id = aws_route53_health_check.primary.id
  
  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

# Secondary endpoint
resource "aws_route53_record" "api_secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  set_identifier = "secondary"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
  
  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}

# Health check
resource "aws_route53_health_check" "primary" {
  fqdn              = aws_lb.primary.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}
```

---

## Communication Plan

### Incident Communication Template

```
SUBJECT: [INCIDENT] Production Service Disruption

STATUS: Investigating / Identified / Monitoring / Resolved

IMPACT:
- Affected Services: [list]
- User Impact: [describe]
- Started At: [timestamp]

CURRENT STATUS:
[Brief description of what's happening]

ACTIONS TAKEN:
1. [action 1]
2. [action 2]

NEXT STEPS:
1. [next step 1]
2. [next step 2]

ETR (Estimated Time to Resolution): [time]

Will provide next update in [timeframe]
```

### Status Page Update

```bash
# Using Atlassian Statuspage API
curl -X POST https://api.statuspage.io/v1/pages/PAGE_ID/incidents \
  -H "Authorization: OAuth YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "incident": {
      "name": "Database Performance Degradation",
      "status": "investigating",
      "impact_override": "minor",
      "body": "We are investigating reports of slow API response times."
    }
  }'
```

---

## DR Checklist

### Before Disaster
- [ ] Terraform state backed up to S3 with versioning
- [ ] Velero backups running successfully
- [ ] RDS automated backups enabled (35 days retention)
- [ ] ECR images replicated to secondary region
- [ ] Secrets replicated to Secrets Manager in secondary region
- [ ] DR runbook documented and tested
- [ ] Contact information up to date
- [ ] Team trained on DR procedures

### During Disaster
- [ ] Incident declared with appropriate severity
- [ ] On-call team mobilized
- [ ] Stakeholders notified
- [ ] Status page updated
- [ ] Recovery procedures initiated
- [ ] Progress communicated regularly

### After Recovery
- [ ] Services fully restored and validated
- [ ] Stakeholders notified of resolution
- [ ] Post-mortem scheduled
- [ ] Root cause identified
- [ ] Action items created
- [ ] DR plan updated based on lessons learned

---

[← Back to Operations Guide](./17-operations-guide.md) | [Next: Cost Analysis →](./19-cost-analysis.md)
