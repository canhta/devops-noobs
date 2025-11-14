# 19. Cost Analysis and Optimization

## Table of Contents
1. [Overview](#overview)
2. [Monthly Cost Breakdown](#monthly-cost-breakdown)
3. [Cost by Environment](#cost-by-environment)
4. [Detailed Component Costs](#detailed-component-costs)
5. [Cost Optimization Strategies](#cost-optimization-strategies)
6. [Cost Comparison Scenarios](#cost-comparison-scenarios)
7. [Scaling Cost Projections](#scaling-cost-projections)
8. [Cost Monitoring and Alerts](#cost-monitoring-and-alerts)
9. [FinOps Best Practices](#finops-best-practices)
10. [Cost Reduction Checklist](#cost-reduction-checklist)

---

## Overview

This document provides a comprehensive cost analysis for the DevOps infrastructure on AWS, including detailed breakdowns, optimization strategies, and scaling projections.

### Cost Philosophy

- **Right-Sizing**: Match resources to actual needs
- **Spot Instances**: Use when appropriate (60-70% savings)
- **Reserved Instances**: Commit for stable workloads (30-40% savings)
- **Auto-Scaling**: Scale down during low usage
- **Resource Lifecycle**: Delete unused resources
- **Monitoring**: Track and optimize continuously

---

## Monthly Cost Breakdown

### Complete Infrastructure Cost (Both Environments)

| Category | Development | Production | Total | % of Total |
|----------|-------------|------------|-------|------------|
| **Compute (EKS)** | $122 | $744 | $866 | 37.8% |
| **Event Streaming (Kafka)** | $81 | $660 | $741 | 32.4% |
| **Database (RDS)** | $15 | $500 | $515 | 22.5% |
| **Networking** | $32 | $96 | $128 | 5.6% |
| **Load Balancing** | $20 | $40 | $60 | 2.6% |
| **Storage (S3, EBS)** | $15 | $35 | $50 | 2.2% |
| **Container Registry (ECR)** | $5 | $10 | $15 | 0.7% |
| **Monitoring & Logs** | $8 | $15 | $23 | 1.0% |
| **DNS (Route53)** | $2 | $3 | $5 | 0.2% |
| **Bastion Host** | $8 | $0 | $8 | 0.3% |
| **Data Transfer** | $10 | $30 | $40 | 1.7% |
| **Misc (KMS, Secrets)** | $5 | $10 | $15 | 0.7% |
| **TOTAL** | **$323** | **$2,143** | **$2,466** | **100%** |

### Visual Cost Distribution

```
Production (87%)  ████████████████████████████████████████████████
Development (13%) ███████
```

---

## Cost by Environment

### Development Environment

**Target**: Cost-effective learning and testing environment

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| **EKS Control Plane** | 1 cluster | $72 |
| **EKS Worker Nodes** | 2x t3.medium spot | $50 |
| **MSK Kafka** | 2 brokers, kafka.t3.small | $61 |
| **MSK Storage** | 200 GB | $20 |
| **RDS PostgreSQL** | db.t3.micro, 20 GB | $15 |
| **NAT Gateway** | 1 gateway (single AZ) | $32 |
| **Application Load Balancer** | 1 ALB, light traffic | $20 |
| **S3 Storage** | ~50 GB | $3 |
| **EBS Volumes** | 150 GB gp3 | $12 |
| **ECR Storage** | 30 GB | $5 |
| **CloudWatch Logs** | 10 GB/month | $8 |
| **Route53 Hosted Zone** | 1 zone | $2 |
| **Bastion Host** | t3.micro | $8 |
| **Data Transfer** | ~100 GB | $10 |
| **KMS, Secrets Manager** | Various | $5 |
| **Subtotal** | | **~$323/month** |

**Cost Optimization Applied**:
- ✅ Spot instances for all workers (70% savings)
- ✅ Single NAT Gateway (saves ~$64/month)
- ✅ Smaller instance types
- ✅ Fewer Kafka brokers
- ✅ Single-AZ RDS
- ✅ Shorter log retention (7 days)

---

### Production Environment

**Target**: High availability, performance, and reliability

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| **EKS Control Plane** | 1 cluster | $72 |
| **EKS Worker Nodes (On-Demand)** | 3x t3.large | $182 |
| **EKS Worker Nodes (Spot)** | 4x t3.large avg | $120 |
| **Auto Scaling Reserve** | Variable capacity | $370 |
| **MSK Kafka** | 3 brokers, kafka.m5.large | $460 |
| **MSK Storage** | 1,500 GB | $150 |
| **MSK Provisioned Throughput** | 250 MB/s per broker | $50 |
| **RDS Aurora** | db.r6g.large, 2 instances | $424 |
| **Aurora Storage** | ~200 GB | $40 |
| **Aurora I/O** | ~10M requests | $20 |
| **NAT Gateway** | 3 gateways (Multi-AZ) | $96 |
| **Application Load Balancer** | 1 ALB, moderate traffic | $40 |
| **S3 Storage** | ~200 GB mixed tiers | $20 |
| **EBS Volumes** | 900 GB gp3 | $75 |
| **RDS Backups** | 200 GB | $20 |
| **ECR Storage** | 60 GB | $10 |
| **CloudWatch Logs** | 50 GB/month | $25 |
| **CloudWatch Metrics** | Custom metrics | $10 |
| **Route53 Hosted Zone** | 1 zone, health checks | $3 |
| **Data Transfer** | ~300 GB out | $30 |
| **KMS Keys** | 3 keys | $9 |
| **Secrets Manager** | 10 secrets | $4 |
| **Subtotal** | | **~$2,190/month** |

**High Availability Features**:
- ✅ Multi-AZ deployment (3 AZs)
- ✅ Mix of on-demand (30%) and spot (70%) instances
- ✅ Aurora with read replica
- ✅ 3 Kafka brokers
- ✅ NAT Gateway per AZ
- ✅ Automated backups with 7-day retention

---

## Detailed Component Costs

### 1. Amazon EKS

#### Control Plane
```
Cost: $0.10/hour per cluster
Monthly: $0.10 × 730 hours = $73/month per cluster

Dev:  1 cluster = $73
Prod: 1 cluster = $73
Total: $146/month
```

#### Worker Nodes - Development

```
Configuration: 2× t3.medium Spot instances
Base hourly rate: $0.0416 (on-demand)
Spot discount: ~70%
Spot effective rate: $0.0125/hour

Monthly cost: 2 × $0.0125 × 730 = $18.25
With auto-scaling overhead: ~$50/month
```

#### Worker Nodes - Production

```
On-Demand (30%): 3× t3.large
Hourly rate: $0.0832
Monthly: 3 × $0.0832 × 730 = $182.21

Spot (70%): 4× t3.large average
Base rate: $0.0832
Spot rate: $0.025/hour (70% off)
Monthly: 4 × $0.025 × 730 = $73

Auto-scaling reserve (peak): ~$370/month
Average: ~$625/month

Production total: $72 (control) + $625 (nodes) = $697/month
```

**Node Sizing Guide**:

| Workload Type | Instance Type | vCPU | RAM | On-Demand | Spot (avg) |
|---------------|---------------|------|-----|-----------|------------|
| Dev/Test | t3.medium | 2 | 4 GB | $30/month | $9/month |
| Small Prod | t3.large | 2 | 8 GB | $61/month | $18/month |
| Medium Prod | t3.xlarge | 4 | 16 GB | $122/month | $37/month |
| Large Prod | t3.2xlarge | 8 | 32 GB | $243/month | $73/month |
| Memory-Optimized | r6g.large | 2 | 16 GB | $88/month | $26/month |
| Compute-Optimized | c6g.large | 2 | 4 GB | $56/month | $17/month |

---

### 2. Amazon MSK (Kafka)

#### Development Configuration

```
2 brokers × kafka.t3.small
Hourly rate: $0.042/broker
Monthly: 2 × $0.042 × 730 = $61.32

Storage: 100 GB per broker = 200 GB total
Storage rate: $0.10/GB-month
Monthly: 200 × $0.10 = $20

Total: $61 + $20 = $81/month
```

#### Production Configuration

```
3 brokers × kafka.m5.large
Hourly rate: $0.21/broker
Monthly: 3 × $0.21 × 730 = $459.90

Storage: 500 GB per broker = 1,500 GB total
Storage rate: $0.10/GB-month
Monthly: 1,500 × $0.10 = $150

Provisioned throughput: 250 MB/s per broker
Rate: $0.0023/MB/s-hour per broker
Monthly: 3 × 250 × $0.0023 × 730 = $1,262.25
Note: Only needed for high-throughput workloads
Without provisioned: Total = $610/month
With provisioned: Total = $1,872/month

Typical setup: $610/month (without provisioned throughput)
High-throughput: $1,872/month (with provisioned throughput)
```

**MSK Instance Comparison**:

| Instance Type | vCPU | Memory | Network | Hourly | Monthly (1 broker) |
|---------------|------|--------|---------|--------|---------------------|
| kafka.t3.small | 2 | 2 GB | Up to 5 Gbps | $0.042 | $31 |
| kafka.m5.large | 2 | 8 GB | Up to 10 Gbps | $0.21 | $153 |
| kafka.m5.xlarge | 4 | 16 GB | Up to 10 Gbps | $0.42 | $307 |
| kafka.m5.2xlarge | 8 | 32 GB | Up to 10 Gbps | $0.84 | $613 |
| kafka.m5.4xlarge | 16 | 64 GB | 10 Gbps | $1.68 | $1,226 |

---

### 3. Amazon RDS/Aurora

#### Development - RDS PostgreSQL

```
Instance: db.t3.micro
Hourly rate: $0.017
Monthly: $0.017 × 730 = $12.41

Storage: 20 GB gp3
Rate: $0.115/GB-month
Monthly: 20 × $0.115 = $2.30

Backup storage: ~20 GB (free up to DB size)
Monthly: $0

Total: $12.41 + $2.30 = $14.71/month
```

#### Production - Aurora PostgreSQL

```
Writer Instance: db.r6g.large
Hourly rate: $0.29
Monthly: $0.29 × 730 = $211.70

Reader Instance: db.r6g.large
Hourly rate: $0.29
Monthly: $0.29 × 730 = $211.70

Storage: ~200 GB
Rate: $0.10/GB-month
Monthly: 200 × $0.10 = $20

I/O: ~10 million requests
Rate: $0.20 per 1M requests
Monthly: 10 × $0.20 = $2

Backup storage: 200 GB
Rate: $0.021/GB-month (beyond DB size)
Monthly: 200 × $0.021 = $4.20

Total: $211.70 + $211.70 + $20 + $2 + $4.20 = $449.60/month
```

**RDS Instance Comparison**:

| Instance Type | vCPU | Memory | On-Demand | 1-Year RI | 3-Year RI | Savings |
|---------------|------|--------|-----------|-----------|-----------|---------|
| db.t3.micro | 2 | 1 GB | $12/mo | $8/mo | $5/mo | 58% |
| db.t3.small | 2 | 2 GB | $25/mo | $17/mo | $11/mo | 56% |
| db.t3.medium | 2 | 4 GB | $50/mo | $34/mo | $22/mo | 56% |
| db.r6g.large | 2 | 16 GB | $212/mo | $134/mo | $85/mo | 60% |
| db.r6g.xlarge | 4 | 32 GB | $424/mo | $268/mo | $170/mo | 60% |

---

### 4. Networking Costs

#### NAT Gateway

```
Development (1 NAT Gateway):
Gateway charge: $0.045/hour
Monthly: $0.045 × 730 = $32.85

Data processing: $0.045/GB
Assuming 100 GB: 100 × $0.045 = $4.50

Total: $32.85 + $4.50 = $37.35/month

Production (3 NAT Gateways):
Gateway charge: 3 × $32.85 = $98.55
Data processing: 300 GB × $0.045 = $13.50

Total: $98.55 + $13.50 = $112.05/month
```

**NAT Gateway Optimization**:
- Dev: Use single NAT Gateway = **Save $65/month**
- Consider VPC endpoints for AWS services = **Save ~$10-20/month**
- Use S3 Gateway endpoint (free) instead of NAT for S3 access

#### VPC Endpoints (Alternative to NAT for AWS services)

```
Interface Endpoint Pricing:
Hourly: $0.01/hour per AZ
Monthly: $0.01 × 730 × 3 AZs = $21.90/endpoint

Data processing: $0.01/GB (first 1 PB)

Common endpoints:
- ecr.api: $22/month
- ecr.dkr: $22/month
- s3: Free (Gateway endpoint)
- logs: $22/month
- secretsmanager: $22/month

Potential savings: Reduce NAT Gateway data processing costs
```

---

### 5. Load Balancing

#### Application Load Balancer

```
Development:
ALB charge: $0.0225/hour
Monthly: $0.0225 × 730 = $16.43

LCU charge: $0.008/LCU-hour
Low traffic ~5 LCUs: 5 × $0.008 × 730 = $29.20

Total: $16.43 + $29.20 = $45.63/month
Typical: ~$20-30/month with minimal traffic

Production:
ALB charge: $16.43
LCU charge (moderate traffic ~10 LCUs): $58.40

Total: $16.43 + $58.40 = $74.83/month
Typical: ~$40-60/month
```

**LCU Calculation** (billed on highest of):
- 25 new connections per second
- 3,000 active connections per minute
- 1 GB per hour for EC2/IP targets
- 0.4 GB per hour for Lambda targets
- 1,000 rule evaluations per second

---

### 6. Storage Costs

#### Amazon S3

```
Development:
Standard storage: 50 GB × $0.023 = $1.15/month
Requests: ~10,000 PUT/POST: 10 × $0.005 = $0.05
Requests: ~50,000 GET: 50 × $0.0004 = $0.02

Total: ~$1.22/month

Production:
Standard: 100 GB × $0.023 = $2.30
Standard-IA: 50 GB × $0.0125 = $0.63
Glacier IR: 50 GB × $0.004 = $0.20
Requests: ~50,000 PUT: 50 × $0.005 = $0.25
Requests: ~500,000 GET: 500 × $0.0004 = $0.20

Total: ~$3.58/month
```

#### EBS Volumes

```
Development:
gp3: 150 GB × $0.08 = $12/month
Snapshots: 50 GB × $0.05 = $2.50/month

Total: $14.50/month

Production:
gp3: 900 GB × $0.08 = $72/month
Snapshots: 300 GB × $0.05 = $15/month

Total: $87/month
```

**Storage Optimization**:
- Use S3 Lifecycle policies to move to cheaper tiers
- Delete old EBS snapshots
- Use gp3 instead of gp2 (20% cheaper, better performance)
- Enable EBS volume auto-delete on instance termination

---

### 7. Container Registry (ECR)

```
Development:
Storage: 30 GB × $0.10 = $3/month
Data transfer out: 10 GB × $0.09 = $0.90

Total: ~$4/month

Production:
Storage: 60 GB × $0.10 = $6/month
Data transfer out: 50 GB × $0.09 = $4.50

Total: ~$10.50/month
```

**ECR Optimization**:
- Implement lifecycle policies (auto-delete old images)
- Keep only last 10 tagged versions
- Delete untagged images after 7 days
- Use image compression

---

### 8. Monitoring and Logging

#### CloudWatch

```
Development:
Log ingestion: 10 GB × $0.50 = $5/month
Log storage: 5 GB × $0.03 = $0.15/month
Metrics: 50 custom × $0.30 = $15/month
(First 10k metrics free)

Total: ~$8/month

Production:
Log ingestion: 50 GB × $0.50 = $25/month
Log storage: 30 GB × $0.03 = $0.90/month
Metrics: 200 custom × $0.30 = $60/month
Dashboards: 3 × $3 = $9/month
Alarms: 20 × $0.10 = $2/month

Total: ~$97/month

Note: Using Loki for logs significantly reduces CloudWatch costs
With Loki: ~$15/month (metrics + alarms only)
```

---

## Cost by Environment

### Alternative: Strimzi (Self-Managed Kafka)

If using Strimzi instead of MSK:

```
Development:
EKS nodes (included in existing count): $0
EBS volumes for Kafka: 200 GB × $0.08 = $16/month

Total: ~$16/month (vs MSK $81/month)
Savings: $65/month

Production:
Additional EKS capacity: 3× t3.large spot = $55/month
EBS volumes: 1,500 GB × $0.08 = $120/month

Total: ~$175/month (vs MSK $660/month)
Savings: $485/month
```

**Tradeoff**: Save ~$550/month total but add operational complexity

---

## Cost Optimization Strategies

### 1. Compute Optimization (EKS)

| Strategy | Implementation | Potential Savings |
|----------|----------------|-------------------|
| **Use Spot Instances** | 70% of workload on spot | 60-70% on compute |
| **Right-Size Instances** | Monitor actual usage, downsize | 20-40% |
| **Cluster Autoscaler** | Scale down during off-hours | 15-30% |
| **Reserved Instances** | 1-year commitment for stable workload | 30-40% |
| **Savings Plans** | Compute Savings Plans | 15-20% |
| **Karpenter** | Better bin-packing | 10-20% |

**Implementation**:
```yaml
# Use spot instances for flexible workloads
nodeSelector:
  capacity-type: spot

tolerations:
  - key: "spot"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

### 2. Kafka Optimization (MSK)

| Strategy | Implementation | Potential Savings |
|----------|----------------|-------------------|
| **Self-Managed (Strimzi)** | Run Kafka on EKS | 60-70% |
| **Right-Size Brokers** | kafka.m5.large → kafka.t3.large | 30-40% |
| **Reduce Broker Count** | 3 brokers → 2 (dev only) | 33% |
| **Disable Provisioned Throughput** | Use only when needed | ~$1,200/month |
| **Optimize Retention** | Reduce from 7 days to 3 days | 40% storage |

### 3. Database Optimization (RDS/Aurora)

| Strategy | Implementation | Potential Savings |
|----------|----------------|-------------------|
| **Reserved Instances** | 1-year or 3-year RI | 30-60% |
| **Right-Size Instances** | db.r6g.xlarge → db.r6g.large | 50% |
| **Aurora Serverless v2** | Pay per ACU (variable workload) | 30-50% |
| **Read Replica Scaling** | Scale up/down based on load | 20-30% |
| **Reduce Backup Retention** | 7 days → 3 days (dev) | Storage costs |

### 4. Networking Optimization

| Strategy | Implementation | Potential Savings |
|----------|----------------|-------------------|
| **Single NAT Gateway (Dev)** | One NAT instead of multi-AZ | $65/month |
| **VPC Endpoints** | For S3, ECR, Secrets Manager | $10-30/month |
| **S3 Gateway Endpoint** | Free alternative to NAT for S3 | $5-15/month |
| **Reduce Cross-AZ Traffic** | Use pod affinity rules | 10-20% |

### 5. Storage Optimization

| Strategy | Implementation | Potential Savings |
|----------|----------------|-------------------|
| **S3 Lifecycle Policies** | Move to IA, Glacier | 60-90% |
| **Delete Old Snapshots** | Automate snapshot cleanup | 30-50% |
| **ECR Lifecycle** | Keep last 10 images only | 40-60% |
| **gp3 Instead of gp2** | Migrate EBS volumes | 20% |
| **EBS Snapshot Lifecycle** | Auto-delete old snapshots | 30-50% |

### 6. Monitoring Optimization

| Strategy | Implementation | Potential Savings |
|----------|----------------|-------------------|
| **Use Loki for Logs** | Instead of CloudWatch Logs | 70-80% |
| **Reduce Log Retention** | 7 days instead of 30 | 75% |
| **Metric Filtering** | Only essential custom metrics | 30-50% |
| **Log Sampling** | Sample high-volume logs | 40-60% |

---

## Cost Comparison Scenarios

### Scenario 1: Startup (Minimal MVP)

**Requirements**: Dev environment only, learning phase

| Component | Configuration | Monthly Cost |
|-----------|--------------|--------------|
| EKS | 1 cluster, 2× t3.small spot | $80 |
| Kafka | Strimzi on EKS | $10 |
| Database | db.t3.micro | $15 |
| Networking | Single NAT | $35 |
| Storage | Minimal | $10 |
| **Total** | | **~$150/month** |

### Scenario 2: Small Team (Dev + Staging)

**Requirements**: Dev + Staging, small traffic

| Component | Configuration | Monthly Cost |
|-----------|--------------|--------------|
| EKS (Dev) | 2× t3.medium spot | $122 |
| EKS (Staging) | 2× t3.large spot | $150 |
| Kafka | MSK dev + staging | $160 |
| Database | RDS t3.small + t3.medium | $80 |
| Networking | 2 environments | $120 |
| Storage & Misc | Standard | $60 |
| **Total** | | **~$692/month** |

### Scenario 3: Production-Ready (Dev + Prod)

**Requirements**: Current design, high availability

| Component | Monthly Cost |
|-----------|--------------|
| As detailed above | **$2,466/month** |

### Scenario 4: Enterprise Scale

**Requirements**: Multi-region, high traffic

| Component | Configuration | Monthly Cost |
|-----------|--------------|--------------|
| EKS (Prod × 2 regions) | Multi-cluster | $1,600 |
| EKS (Dev) | Single region | $122 |
| Kafka (Prod × 2 regions) | Multi-region MSK | $1,400 |
| Database | Aurora Global Database | $1,200 |
| Networking | Multi-region | $300 |
| CDN (CloudFront) | High traffic | $500 |
| WAF | Protection | $80 |
| Storage & Misc | Increased | $300 |
| **Total** | | **~$5,502/month** |

---

## Scaling Cost Projections

### 6-Month Projection

Assuming gradual traffic growth:

| Component | Current | 6 Months | Increase |
|-----------|---------|----------|----------|
| EKS Nodes (Prod) | $625 | $950 | +52% |
| Kafka (Prod) | $660 | $660 | 0% |
| Database (Prod) | $450 | $900 | +100%* |
| Data Transfer | $30 | $80 | +167% |
| Storage | $35 | $60 | +71% |
| **Total Production** | **$2,143** | **$3,110** | **+45%** |

*Database scaling might require instance upsize or read replicas

### 1-Year Projection

| Component | Current | 1 Year | Strategy |
|-----------|---------|--------|----------|
| EKS Nodes | $625 | $1,400 | Auto-scaling + RI |
| Kafka | $660 | $1,100 | Add brokers or optimize |
| Database | $450 | $1,200 | Aurora, possible sharding |
| Everything Else | $408 | $800 | Proportional growth |
| **Total Production** | **$2,143** | **$4,500** | **+110%** |

### 2-Year Projection (Enterprise Scale)

- Multi-region deployment
- Database sharding
- CDN integration
- Advanced observability
- **Estimated**: $8,000-12,000/month

---

## Cost Monitoring and Alerts

### AWS Cost Explorer Setup

```bash
# Enable Cost Allocation Tags
aws ce put-cost-categories --name Environment \
  --rules '[{"Value":"dev","Rule":{"Tags":{"Key":"Environment","Values":["dev"]}}}]'

# Tag all resources
default_tags {
  Environment = var.environment
  Project     = "DevOps Platform"
  CostCenter  = "Engineering"
}
```

### Cost Anomaly Detection

```hcl
resource "aws_ce_anomaly_monitor" "service" {
  name              = "ServiceMonitor"
  monitor_type      = "DIMENSIONAL"
  monitor_dimension = "SERVICE"
}

resource "aws_ce_anomaly_subscription" "subscription" {
  name      = "CostAnomalySubscription"
  frequency = "DAILY"

  monitor_arn_list = [
    aws_ce_anomaly_monitor.service.arn
  ]

  subscriber {
    type    = "EMAIL"
    address = "devops-team@example.com"
  }

  threshold_expression {
    dimension {
      key           = "ANOMALY_TOTAL_IMPACT_ABSOLUTE"
      values        = ["100"]
      match_options = ["GREATER_THAN_OR_EQUAL"]
    }
  }
}
```

### Budget Alerts

```hcl
resource "aws_budgets_budget" "monthly" {
  name              = "monthly-infrastructure-budget"
  budget_type       = "COST"
  limit_amount      = "3000"
  limit_unit        = "USD"
  time_period_start = "2024-01-01_00:00"
  time_unit         = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "FORECASTED"
    subscriber_email_addresses = ["devops-team@example.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = ["devops-team@example.com"]
  }
}
```

---

## FinOps Best Practices

### 1. Tagging Strategy

```hcl
# Apply to all resources
tags = {
  Environment = "production"
  Project     = "DevOps Platform"
  Team        = "Platform Engineering"
  CostCenter  = "Engineering"
  Owner       = "devops-team@example.com"
  Application = "api-gateway"
  ManagedBy   = "Terraform"
}
```

### 2. Regular Cost Reviews

**Weekly**:
- Review AWS Cost Explorer for anomalies
- Check for unused resources

**Monthly**:
- Detailed cost breakdown by service
- Evaluate Reserved Instance opportunities
- Review and optimize storage costs

**Quarterly**:
- Re-evaluate architecture decisions
- Consider new AWS services/features
- Benchmark against industry standards

### 3. Automated Cost Optimization

```python
# Lambda function to stop dev resources after hours
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Stop dev instances during off-hours (6 PM - 8 AM)
    instances = ec2.describe_instances(
        Filters=[
            {'Name': 'tag:Environment', 'Values': ['dev']},
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )
    
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            ec2.stop_instances(InstanceIds=[instance['InstanceId']])
    
    return {'statusCode': 200, 'body': 'Dev instances stopped'}
```

### 4. Cost Attribution

Track costs by:
- **Environment** (dev, staging, prod)
- **Team** (platform, application, data)
- **Application** (service-specific costs)
- **Feature** (experimental vs stable)

---

## Cost Reduction Checklist

### Immediate Actions (0-1 Week)

- [ ] Enable AWS Cost Anomaly Detection
- [ ] Set up budget alerts
- [ ] Tag all resources with Environment, Project, Team
- [ ] Identify and delete unused EBS volumes
- [ ] Identify and delete unused EBS snapshots
- [ ] Review and optimize CloudWatch Log retention
- [ ] Delete old ECR images (set lifecycle policies)
- [ ] Implement S3 lifecycle policies

**Potential Savings**: 10-15% ($200-300/month)

### Short-Term Actions (1-4 Weeks)

- [ ] Migrate to Spot instances for flexible workloads
- [ ] Implement Cluster Autoscaler
- [ ] Right-size EKS instances based on actual usage
- [ ] Evaluate Strimzi vs MSK for Kafka
- [ ] Reduce NAT Gateway count in dev (single NAT)
- [ ] Implement VPC endpoints for common services
- [ ] Review and optimize RDS instance sizes
- [ ] Enable gp3 for all EBS volumes

**Potential Savings**: 20-30% ($400-600/month)

### Medium-Term Actions (1-3 Months)

- [ ] Purchase Reserved Instances for stable workloads
- [ ] Migrate logs to Loki (reduce CloudWatch costs)
- [ ] Implement Karpenter for better bin-packing
- [ ] Optimize Kafka topic retention and partition count
- [ ] Implement Aurora Serverless v2 if applicable
- [ ] Set up automated start/stop for dev resources
- [ ] Optimize data transfer costs

**Potential Savings**: 30-40% ($600-800/month)

### Long-Term Actions (3-6 Months)

- [ ] Consider Savings Plans
- [ ] Evaluate multi-year Reserved Instances
- [ ] Implement advanced auto-scaling strategies
- [ ] Consider AWS Graviton instances (ARM-based)
- [ ] Optimize application code for resource efficiency
- [ ] Implement cost allocation and chargeback

**Potential Savings**: 40-50% ($800-1,000/month)

---

## Summary

### Total Cost of Ownership (TCO)

| Scenario | Monthly | Annual | 3-Year |
|----------|---------|--------|--------|
| **Minimal MVP** | $150 | $1,800 | $5,400 |
| **Current (Dev + Prod)** | $2,466 | $29,592 | $88,776 |
| **Optimized (Dev + Prod)** | $1,800 | $21,600 | $64,800 |
| **Enterprise Scale** | $5,500 | $66,000 | $198,000 |

### Cost Breakdown by Category

```
Compute (38%)     ████████████████████
Kafka (32%)       █████████████████
Database (23%)    ████████████
Networking (6%)   ███
Other (1%)        █
```

### Top 3 Cost Drivers

1. **Compute (EKS)**: 38% - Optimize with spot instances, right-sizing, autoscaling
2. **Kafka (MSK)**: 32% - Consider Strimzi for 60% savings
3. **Database (RDS)**: 23% - Use Reserved Instances for 40% savings

### Optimization Priority

**High Impact, Easy to Implement**:
1. Spot instances for EKS workers → Save $300-400/month
2. Single NAT Gateway in dev → Save $65/month
3. ECR/EBS lifecycle policies → Save $50-100/month
4. Loki instead of CloudWatch Logs → Save $80/month

**High Impact, Medium Effort**:
1. Strimzi instead of MSK → Save $550/month
2. Reserved Instances → Save $400-600/month
3. Cluster Autoscaler → Save $200-400/month

**Ongoing Optimization**:
1. Regular cost reviews
2. Resource right-sizing
3. Unused resource cleanup
4. Continuous monitoring and alerts

---

**Next Steps**: 
1. Implement immediate cost optimizations
2. Set up cost monitoring and alerts
3. Evaluate Reserved Instance opportunities
4. Create monthly cost review process

---

[← Back to Disaster Recovery](./18-disaster-recovery.md) | [Next: Step-by-Step Implementation →](./20-step-by-step-implementation.md)
