# 17. Operations Guide

## Table of Contents
1. [Overview](#overview)
2. [Daily Operations](#daily-operations)
3. [Monitoring and Alerting](#monitoring-and-alerting)
4. [Backup and Restore](#backup-and-restore)
5. [Incident Response](#incident-response)
6. [Maintenance Windows](#maintenance-windows)
7. [Scaling Operations](#scaling-operations)
8. [Security Operations](#security-operations)
9. [Troubleshooting Runbooks](#troubleshooting-runbooks)
10. [On-Call Procedures](#on-call-procedures)

---

## Overview

This document provides comprehensive operational procedures for managing the DevOps platform on a day-to-day basis, including monitoring, maintenance, incident response, and troubleshooting.

### Operational Responsibilities

**Daily**:
- Monitor dashboards and alerts
- Review logs for errors
- Check resource utilization
- Verify backup completion

**Weekly**:
- Review cost reports
- Update security patches
- Test disaster recovery procedures
- Audit access logs

**Monthly**:
- Capacity planning review
- Security audit
- Update documentation
- Performance optimization

### Operations Team Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    Operations Team                           │
├─────────────────────────────────────────────────────────────┤
│  Platform Team        │  Responsibilities                    │
│  - Infrastructure     │  - AWS resources                    │
│  - Kubernetes         │  - EKS clusters                     │
│  - Networking         │  - VPC, Security Groups             │
├─────────────────────────────────────────────────────────────┤
│  Application Team     │  Responsibilities                    │
│  - Deployments        │  - CI/CD pipelines                  │
│  - Monitoring         │  - Application health               │
│  - Incidents          │  - Bug fixes                        │
├─────────────────────────────────────────────────────────────┤
│  Data Team            │  Responsibilities                    │
│  - Databases          │  - RDS/Aurora                       │
│  - Kafka              │  - Event streaming                  │
│  - Backups            │  - Data retention                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Daily Operations

### Morning Checklist

```bash
#!/bin/bash
# daily-health-check.sh

echo "=== Daily Health Check ==="
echo ""

# Check cluster status
echo "1. Cluster Status"
kubectl cluster-info
kubectl get nodes
echo ""

# Check pod health
echo "2. Pod Health (Non-Running Pods)"
kubectl get pods -A | grep -v Running | grep -v Completed
echo ""

# Check recent pod restarts
echo "3. Recent Restarts (last 24h)"
kubectl get pods -A -o json | jq -r '.items[] | select(.status.containerStatuses[]?.restartCount > 0) | "\(.metadata.namespace)/\(.metadata.name): \(.status.containerStatuses[0].restartCount) restarts"'
echo ""

# Check PVC status
echo "4. PVC Status"
kubectl get pvc -A | grep -v Bound
echo ""

# Check cert-manager certificates
echo "5. Certificate Status"
kubectl get certificates -A | grep -v True
echo ""

# Check HPA status
echo "6. HPA Status"
kubectl get hpa -A
echo ""

# Check recent events (errors/warnings)
echo "7. Recent Cluster Events (Last 30 minutes)"
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
echo ""

echo "=== Health Check Complete ==="
```

### Monitoring Dashboard Review

**Daily Checks**:
1. Open Grafana dashboards
2. Review Kubernetes cluster overview
3. Check application metrics (request rate, latency, errors)
4. Review resource utilization (CPU, memory, disk)
5. Check alert history

**Key Metrics to Monitor**:
- Request rate and latency (p50, p95, p99)
- Error rate (< 1% target)
- Pod restart count
- Node resource utilization
- Database connections
- Kafka consumer lag

---

## Backup and Restore

### Velero Backup Setup

```bash
# Install Velero
velero install \
  --provider aws \
  --bucket eks-backup-bucket \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero
```

### Automated Backup Schedule

```yaml
# velero-backup-schedule.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  template:
    includedNamespaces:
    - production
    - staging
    excludedResources:
    - events
    - events.events.k8s.io
    ttl: 720h0m0s  # 30 days retention
```

### Manual Backup

```bash
# Backup entire cluster
velero backup create full-backup-$(date +%Y%m%d)

# Backup specific namespace
velero backup create prod-backup --include-namespaces production

# Backup with label selector
velero backup create app-backup --selector app=api-gateway
```

### Restore from Backup

```bash
# List backups
velero backup get

# Restore from specific backup
velero restore create --from-backup daily-backup-20231115

# Restore specific namespace
velero restore create --from-backup full-backup --include-namespaces production
```

### Database Backup

```bash
# RDS automated backups (configured in Terraform)
# Manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier prod-db \
  --db-snapshot-identifier manual-snapshot-$(date +%Y%m%d)

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-db-restored \
  --db-snapshot-identifier manual-snapshot-20231115
```

---

## Incident Response

### Incident Severity Levels

| Severity | Impact | Response Time | Examples |
|----------|--------|---------------|----------|
| **P1 - Critical** | Service down | < 15 min | Complete outage, data loss |
| **P2 - High** | Major degradation | < 1 hour | High error rates, slow response |
| **P3 - Medium** | Partial impact | < 4 hours | Single service affected |
| **P4 - Low** | Minor impact | Next business day | Cosmetic issues, logging errors |

### Incident Response Process

```
1. DETECT
   └─> Alert triggers or user report

2. ACKNOWLEDGE
   └─> On-call engineer acknowledges
   └─> Create incident ticket

3. ASSESS
   └─> Determine severity
   └─> Check dashboards and logs
   └─> Identify impact

4. COMMUNICATE
   └─> Notify stakeholders
   └─> Create status page update

5. MITIGATE
   └─> Implement temporary fix
   └─> Roll back if needed

6. RESOLVE
   └─> Implement permanent fix
   └─> Verify resolution
   └─> Update status page

7. POST-MORTEM
   └─> Document incident
   └─> Identify root cause
   └─> Create action items
```

### Common Incident Runbooks

#### High Error Rate

```bash
# 1. Check error rate
kubectl logs -n production -l app=api-gateway --tail=100 | grep ERROR

# 2. Check recent deployments
kubectl rollout history deployment/api-gateway -n production

# 3. Roll back if needed
kubectl rollout undo deployment/api-gateway -n production

# 4. Check dependencies
kubectl get pods -n production
kubectl logs -n production -l app=database-service --tail=50
```

#### Pod CrashLooping

```bash
# 1. Identify crashing pods
kubectl get pods -n production | grep CrashLoopBackOff

# 2. Check logs
kubectl logs <pod-name> -n production --previous

# 3. Describe pod for events
kubectl describe pod <pod-name> -n production

# 4. Check resource limits
kubectl top pod <pod-name> -n production

# 5. Check recent config changes
kubectl get configmap -n production
kubectl get secret -n production
```

---

## Maintenance Windows

### Scheduled Maintenance Process

**1 Week Before**:
- Announce maintenance window
- Review change plan
- Prepare rollback procedure
- Test changes in staging

**Day Before**:
- Re-announce maintenance
- Backup critical data
- Verify rollback readiness

**During Maintenance**:
```bash
# 1. Set maintenance mode (drain traffic)
kubectl cordon <node-name>

# 2. Perform updates
helm upgrade <release> <chart> -n <namespace>

# 3. Monitor deployment
kubectl rollout status deployment/<name> -n <namespace>

# 4. Run smoke tests
curl -f https://api.example.com/health

# 5. Re-enable traffic
kubectl uncordon <node-name>
```

### EKS Cluster Upgrade

```bash
# 1. Check current version
kubectl version --short

# 2. Review upgrade guide
aws eks describe-addon-versions --kubernetes-version 1.28

# 3. Upgrade control plane
aws eks update-cluster-version \
  --name eks-prod \
  --kubernetes-version 1.28

# 4. Wait for upgrade to complete
aws eks describe-cluster --name eks-prod --query cluster.status

# 5. Update node groups
aws eks update-nodegroup-version \
  --cluster-name eks-prod \
  --nodegroup-name prod-nodes \
  --kubernetes-version 1.28

# 6. Update add-ons
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.15/config/master/aws-k8s-cni.yaml
```

---

## Security Operations

### Security Audit Checklist

```bash
# Check for privileged pods
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.privileged == true) | {namespace: .metadata.namespace, pod: .metadata.name}'

# Check for pods running as root
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.runAsUser == 0 or .spec.securityContext.runAsUser == 0) | {namespace: .metadata.namespace, pod: .metadata.name}'

# Check for default service accounts in use
kubectl get pods -A -o json | jq '.items[] | select(.spec.serviceAccountName == "default") | {namespace: .metadata.namespace, pod: .metadata.name}'

# Check exposed services
kubectl get svc -A | grep LoadBalancer

# Review RBAC bindings
kubectl get clusterrolebindings -o json | jq '.items[] | select(.subjects[]?.name == "system:anonymous") | .metadata.name'
```

### Certificate Rotation

```bash
# Check certificate expiration
kubectl get certificates -A

# Force renewal
kubectl delete certificaterequest -n production <cert-request-name>

# Verify new certificate
kubectl describe certificate -n production <cert-name>
```

---

## On-Call Procedures

### On-Call Handoff Checklist

- [ ] Review open incidents
- [ ] Review recent changes/deployments
- [ ] Check alert status
- [ ] Review upcoming maintenance
- [ ] Ensure access to all systems
- [ ] Test paging/alerting

### Emergency Contacts

```
Incident Severity P1:
- Engineering Lead: +1-XXX-XXX-XXXX
- DevOps Lead: +1-XXX-XXX-XXXX
- CTO: +1-XXX-XXX-XXXX

Vendor Support:
- AWS Support: Case via Console
- Database Support: support@vendor.com
```

---

[← Back to Performance Optimization](./16-performance-optimization.md) | [Next: Disaster Recovery →](./18-disaster-recovery.md)
