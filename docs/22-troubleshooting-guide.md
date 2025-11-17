# 22. Troubleshooting Guide

## Table of Contents
1. [Overview](#overview)
2. [Debugging Methodology](#debugging-methodology)
3. [EKS Cluster Issues](#eks-cluster-issues)
4. [Pod and Container Issues](#pod-and-container-issues)
5. [Networking Issues](#networking-issues)
6. [Storage Issues](#storage-issues)
7. [Application Issues](#application-issues)
8. [Kafka Issues](#kafka-issues)
9. [Database Issues](#database-issues)
10. [Common Issues and Solutions](#common-issues-and-solutions)

---

## Overview

This comprehensive troubleshooting guide helps diagnose and resolve common issues in the DevOps platform, from infrastructure to application level.

### Debugging Methodology

```
1. Identify Symptoms
   └─> What is the observable problem?

2. Gather Information
   └─> Logs, metrics, events

3. Form Hypothesis
   └─> What could cause this?

4. Test Hypothesis
   └─> Try potential fixes

5. Implement Solution
   └─> Apply the fix

6. Document
   └─> Update runbooks
```

### Essential Tools

```bash
# Kubernetes debugging
kubectl get pods -A
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
kubectl exec -it <pod-name> -- /bin/sh

# Cluster information
kubectl cluster-info
kubectl get nodes -o wide
kubectl top nodes
kubectl top pods -A

# Network debugging
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- /bin/bash

# AWS debugging
aws eks describe-cluster --name <cluster-name>
aws ec2 describe-instances
aws logs tail <log-group> --follow
```

---

## Debugging Methodology

### The 5 Whys

```
Problem: API is returning 500 errors

Why 1: Why are we getting 500 errors?
└─> Database connection failing

Why 2: Why is database connection failing?
└─> Connection pool exhausted

Why 3: Why is connection pool exhausted?
└─> Too many long-running queries

Why 4: Why are there long-running queries?
└─> Missing database index

Why 5: Why is the index missing?
└─> Recent migration didn't include index creation

ROOT CAUSE: Missing database index after migration
SOLUTION: Add missing index, review migration process
```

---

## EKS Cluster Issues

### Cluster Not Accessible

```bash
# Check cluster status
aws eks describe-cluster --name eks-prod --query 'cluster.status'

# Update kubeconfig
aws eks update-kubeconfig --name eks-prod --region us-east-1

# Verify credentials
aws sts get-caller-identity

# Check aws-auth ConfigMap
kubectl get configmap aws-auth -n kube-system -o yaml

# Common fix: Update aws-auth to include your IAM role
kubectl edit configmap aws-auth -n kube-system
```

### Nodes Not Ready

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check node logs (SSH to node or use SSM)
aws ssm start-session --target <instance-id>
journalctl -u kubelet -f

# Common issues:
# 1. Network plugin issues
kubectl logs -n kube-system -l k8s-app=aws-node

# 2. Disk pressure
kubectl top node <node-name>
df -h  # on the node

# 3. Memory pressure
free -h  # on the node

# Fix: Cordon and drain problematic node
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
# Terminate instance to force recreation
aws ec2 terminate-instances --instance-ids <instance-id>
```

---

## Pod and Container Issues

### Pod Stuck in Pending

```bash
# Describe pod to see events
kubectl describe pod <pod-name> -n <namespace>

# Common causes and solutions:

# 1. Insufficient resources
kubectl top nodes
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
# Solution: Add more nodes or reduce resource requests

# 2. PVC not bound
kubectl get pvc -n <namespace>
# Solution: Check storage class and provisioner

# 3. Node affinity/taints
kubectl get nodes --show-labels
kubectl describe node <node-name> | grep Taints
# Solution: Adjust pod affinity or node taints

# 4. Image pull issues
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 Events
# Solution: Check ECR permissions and image name
```

### Pod CrashLoopBackOff

```bash
# View current logs
kubectl logs <pod-name> -n <namespace>

# View previous container logs
kubectl logs <pod-name> -n <namespace> --previous

# Check for common issues:

# 1. Application errors
kubectl logs <pod-name> -n <namespace> --tail=100 | grep -i error

# 2. Missing environment variables
kubectl describe pod <pod-name> -n <namespace> | grep -A 20 "Environment"

# 3. Failed health checks
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Liveness\|Readiness"

# 4. Permission issues
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 5 securityContext

# Debug with interactive shell
kubectl run debug-pod --image=busybox --rm -it --restart=Never -- sh

# Debug existing pod
kubectl debug <pod-name> -n <namespace> -it --image=busybox
```

### OOMKilled Pods

```bash
# Check pod events
kubectl describe pod <pod-name> -n <namespace> | grep -i oom

# Check current memory usage
kubectl top pod <pod-name> -n <namespace>

# View memory limits
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A 5 resources

# Solution: Increase memory limits
kubectl set resources deployment/<deployment-name> \
  -c=<container-name> \
  --limits=memory=1Gi \
  -n <namespace>

# Or use VPA for recommendations
kubectl describe vpa <vpa-name> -n <namespace>
```

---

## Networking Issues

### Service Not Accessible

```bash
# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Check pod labels match service selector
kubectl get pods -n <namespace> --show-labels
kubectl get svc <service-name> -n <namespace> -o yaml | grep selector

# Test service from within cluster
kubectl run test-pod --image=nicolaka/netshoot --rm -it -- /bin/bash
curl http://<service-name>.<namespace>.svc.cluster.local

# Check DNS resolution
nslookup <service-name>.<namespace>.svc.cluster.local

# Check CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Ingress/Load Balancer Issues

```bash
# Check ingress status
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>

# Check ALB Controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Verify security groups
aws ec2 describe-security-groups \
  --filters "Name=tag:kubernetes.io/cluster/<cluster-name>,Values=owned"

# Check target group health
ALB_ARN=$(kubectl get ingress <ingress-name> -n <namespace> -o json | jq -r '.status.loadBalancer.ingress[0].hostname')
TG_ARN=$(aws elbv2 describe-target-groups --load-balancer-arn $ALB_ARN --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

### Network Policy Blocking Traffic

```bash
# List network policies
kubectl get networkpolicies -A

# Describe specific policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# Test connectivity
kubectl run test-source --image=nicolaka/netshoot --rm -it -n <source-namespace> -- /bin/bash
curl http://<service-name>.<target-namespace>.svc.cluster.local

# Temporarily remove network policy for testing
kubectl delete networkpolicy <policy-name> -n <namespace>
# Remember to recreate after testing!
```

---

## Application Issues

### High Latency

```bash
# Check pod CPU/memory
kubectl top pods -n <namespace>

# Check application metrics
curl http://<pod-ip>:3000/metrics | grep http_request_duration

# Check database connection pool
kubectl logs <pod-name> -n <namespace> | grep "connection pool"

# Check external dependencies
kubectl exec <pod-name> -n <namespace> -- curl -w "@curl-format.txt" -o /dev/null -s http://external-api.com

# Analyze with Prometheus
# Query: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
```

### High Error Rate

```bash
# Check recent logs for errors
kubectl logs <pod-name> -n <namespace> --tail=100 | grep -i error

# Check multiple pods
for pod in $(kubectl get pods -n <namespace> -l app=<app-name> -o name); do
  echo "\n=== $pod ==="
  kubectl logs $pod -n <namespace> --tail=20 | grep -i error
done

# Check recent deployments
kubectl rollout history deployment/<deployment-name> -n <namespace>

# Rollback if needed
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# Check Prometheus for error rate
# Query: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

---

## Kafka Issues

### Consumer Lag

```bash
# Check consumer group lag (Strimzi)
kubectl exec -n kafka kafka-cluster-kafka-0 -- bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group <consumer-group>

# Check MSK lag
aws kafka describe-cluster --cluster-arn <cluster-arn>

# Common causes:
# 1. Slow processing - scale consumers
kubectl scale deployment/<consumer-deployment> --replicas=5 -n <namespace>

# 2. Consumer errors - check logs
kubectl logs -n <namespace> -l app=<consumer-app> --tail=100

# 3. Rebalancing - check consumer group
kubectl logs <pod-name> -n <namespace> | grep rebalance
```

### Connection Issues

```bash
# Test Kafka connectivity
kubectl run kafka-test --image=confluentinc/cp-kafka:latest --rm -it --restart=Never -- bash
kafka-broker-api-versions --bootstrap-server kafka-cluster-kafka-bootstrap:9092

# Check Kafka broker status
kubectl get pods -n kafka -l app.kubernetes.io/name=kafka

# Check Kafka logs
kubectl logs -n kafka kafka-cluster-kafka-0 --tail=100

# Verify security groups (MSK)
aws kafka describe-cluster --cluster-arn <arn> --query 'ClusterInfo.BrokerNodeGroupInfo.SecurityGroups'
```

---

## Database Issues

### Connection Pool Exhausted

```bash
# Check application logs
kubectl logs <pod-name> -n <namespace> | grep "connection pool"

# Check RDS connections
aws rds describe-db-instances \
  --db-instance-identifier prod-db \
  --query 'DBInstances[0].DBInstanceStatus'

# Check CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=prod-db \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average

# Solution: Increase max_connections or optimize connection usage
```

### Slow Queries

```bash
# Enable slow query log (RDS)
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-params \
  --parameters "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate"

# Check slow query log
aws rds download-db-log-file-portion \
  --db-instance-identifier prod-db \
  --log-file-name slowquery/postgresql.log

# Connect to database and check
kubectl run psql-client --image=postgres:14 --rm -it --restart=Never -- \
  psql -h <db-host> -U <username> -d <database>

# Check running queries
SELECT pid, now() - query_start as duration, query 
FROM pg_stat_activity 
WHERE state = 'active' AND now() - query_start > interval '1 minute';

# Check missing indexes
SELECT schemaname, tablename, attname, n_distinct, correlation 
FROM pg_stats 
WHERE schemaname NOT IN ('pg_catalog', 'information_schema');
```

---

## Common Issues and Solutions

### Issue: ImagePullBackOff

**Symptoms**: Pod stuck in ImagePullBackOff state

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 Events
```

**Common Causes**:
1. Image doesn't exist
2. ECR permissions missing
3. Wrong image tag
4. Network issues

**Solution**:
```bash
# Verify image exists
aws ecr describe-images --repository-name <repo-name>

# Check IRSA permissions
kubectl get sa <service-account> -n <namespace> -o yaml

# Verify image pull secret (if using)
kubectl get secret <secret-name> -n <namespace>

# Fix IAM permissions
aws iam attach-role-policy \
  --role-name <node-role> \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

### Issue: Certificate Expired

**Symptoms**: HTTPS errors, browser warnings

**Diagnosis**:
```bash
kubectl get certificates -A
kubectl describe certificate <cert-name> -n <namespace>
```

**Solution**:
```bash
# Force renewal
kubectl delete certificaterequest -n <namespace> <cert-request>

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Manually request certificate
kubectl cert-manager renew <certificate-name> -n <namespace>
```

### Issue: High Memory Usage

**Symptoms**: Pods OOMKilled, slow performance

**Diagnosis**:
```bash
kubectl top pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace> | grep -i memory
```

**Solution**:
```bash
# Check for memory leaks in application
kubectl exec <pod-name> -n <namespace> -- ps aux

# Increase memory limits
kubectl set resources deployment/<name> \
  --limits=memory=2Gi \
  --requests=memory=1Gi

# Use VPA for automatic sizing
kubectl apply -f vpa-config.yaml
```

---

[← Back to Bastion Host Setup](./21-bastion-host-setup.md) | [Next: Back to README →](../README.md)
