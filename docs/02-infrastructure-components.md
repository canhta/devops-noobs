# 02. Infrastructure Components

## Table of Contents
1. [Overview](#overview)
2. [Amazon EKS (Elastic Kubernetes Service)](#amazon-eks-elastic-kubernetes-service)
3. [Amazon MSK (Managed Streaming for Apache Kafka)](#amazon-msk-managed-streaming-for-apache-kafka)
4. [Alternative: Self-Managed Kafka (Strimzi)](#alternative-self-managed-kafka-strimzi)
5. [Networking Infrastructure](#networking-infrastructure)
6. [Compute Resources](#compute-resources)
7. [Amazon ECR (Elastic Container Registry)](#amazon-ecr-elastic-container-registry)
8. [Storage Solutions](#storage-solutions)
9. [Database Services](#database-services)
10. [Load Balancing](#load-balancing)
11. [DNS and Domain Management](#dns-and-domain-management)
12. [Infrastructure Comparison Tables](#infrastructure-comparison-tables)

---

## Overview

This document provides detailed specifications for all AWS infrastructure components used in the DevOps platform. Each section includes configuration details, sizing recommendations, and cost considerations for both Dev and Production environments.

### Environment Strategy

| Aspect | Development | Production |
|--------|-------------|------------|
| Purpose | Testing, experimentation | Live customer traffic |
| Availability | Single-AZ acceptable | Multi-AZ required |
| Scaling | Minimal resources | Auto-scaling enabled |
| Cost Priority | Cost optimization | Reliability first |
| Data Retention | Short (7-14 days) | Long (30-90 days) |

---

## Amazon EKS (Elastic Kubernetes Service)

### Overview

Amazon EKS is a managed Kubernetes service that eliminates the need to manage the Kubernetes control plane. AWS handles the control plane's availability, scaling, and patching.

### Cluster Configuration

#### Production Cluster

```hcl
# Terraform example
module "eks_production" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "production-eks-cluster"
  cluster_version = "1.28"

  # Control plane configuration
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true
  
  # Enhanced security
  cluster_encryption_config = {
    resources        = ["secrets"]
    provider_key_arn = aws_kms_key.eks.arn
  }

  # Subnet configuration
  vpc_id     = module.vpc_production.vpc_id
  subnet_ids = module.vpc_production.private_subnets
  
  # Control plane subnet IDs (for ENIs)
  control_plane_subnet_ids = module.vpc_production.control_plane_subnets

  # Cluster addons
  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
    aws-ebs-csi-driver = {
      most_recent = true
    }
  }

  # Node groups configuration
  eks_managed_node_groups = {
    # On-demand node group for critical workloads
    critical_workloads = {
      name           = "critical-ng"
      instance_types = ["t3.large", "t3a.large"]
      capacity_type  = "ON_DEMAND"
      
      min_size     = 2
      max_size     = 10
      desired_size = 3

      labels = {
        workload-type = "critical"
        capacity-type = "on-demand"
      }

      taints = []
      
      disk_size = 100
      disk_type = "gp3"
    }

    # Spot instances for flexible workloads
    flexible_workloads = {
      name           = "flexible-ng"
      instance_types = ["t3.large", "t3a.large", "t2.large"]
      capacity_type  = "SPOT"
      
      min_size     = 2
      max_size     = 20
      desired_size = 4

      labels = {
        workload-type = "flexible"
        capacity-type = "spot"
      }

      taints = [{
        key    = "spot"
        value  = "true"
        effect = "NoSchedule"
      }]
      
      disk_size = 100
      disk_type = "gp3"
    }
  }

  tags = {
    Environment = "production"
    Terraform   = "true"
    ManagedBy   = "terraform"
  }
}
```

#### Development Cluster

```hcl
module "eks_development" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "dev-eks-cluster"
  cluster_version = "1.28"

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  vpc_id     = module.vpc_dev.vpc_id
  subnet_ids = module.vpc_dev.private_subnets

  # Smaller node groups for dev
  eks_managed_node_groups = {
    general = {
      name           = "dev-general-ng"
      instance_types = ["t3.medium"]
      capacity_type  = "SPOT"  # Cost optimization
      
      min_size     = 1
      max_size     = 5
      desired_size = 2

      labels = {
        environment = "dev"
      }
    }
  }

  tags = {
    Environment = "development"
    Terraform   = "true"
  }
}
```

### EKS Specifications

| Component | Development | Production |
|-----------|-------------|------------|
| **Control Plane** |
| Version | 1.28 | 1.28 |
| Endpoint Access | Public + Private | Public + Private |
| Secrets Encryption | Optional | KMS Enabled |
| **Worker Nodes** |
| On-Demand Nodes | 0 | 2-10 (t3.large) |
| Spot Nodes | 1-5 (t3.medium) | 2-20 (t3.large) |
| Disk Size | 50 GB | 100 GB |
| Disk Type | gp3 | gp3 |
| **Networking** |
| CNI | AWS VPC CNI | AWS VPC CNI |
| Network Policy | Calico (optional) | Calico (recommended) |
| Pod CIDR | Auto-assigned | Auto-assigned |
| **Add-ons** |
| CoreDNS | ✅ | ✅ |
| kube-proxy | ✅ | ✅ |
| VPC CNI | ✅ | ✅ |
| EBS CSI Driver | ✅ | ✅ |
| Load Balancer Controller | ✅ | ✅ |

### EKS Features

#### 1. **Control Plane**
- Highly available across multiple AZs
- Automatic version upgrades (scheduled)
- Integrated with AWS CloudWatch for logging
- API server rate limiting

#### 2. **IRSA (IAM Roles for Service Accounts)**
```yaml
# ServiceAccount with IAM role
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/app-role
```

Benefits:
- Pod-level AWS permissions
- No need to store AWS credentials
- Automatic credential rotation
- Fine-grained access control

#### 3. **Node Groups**
- Managed node groups with automated AMI updates
- Support for On-Demand and Spot instances
- Automatic registration with control plane
- Integration with Auto Scaling Groups

#### 4. **Add-ons**
- Managed updates for critical components
- Compatibility guarantee with cluster version
- Simplified upgrade process

### EKS Best Practices

1. **Multi-AZ Deployment**: Always deploy nodes across at least 2 AZs
2. **Resource Quotas**: Set namespace resource quotas to prevent resource exhaustion
3. **Pod Disruption Budgets**: Ensure service availability during node updates
4. **Regular Updates**: Keep cluster and node groups up to date
5. **RBAC**: Implement least-privilege access control
6. **Network Policies**: Restrict pod-to-pod communication
7. **Monitoring**: Enable control plane logging to CloudWatch

### Cost Optimization

| Strategy | Savings | Impact |
|----------|---------|--------|
| Use Spot instances | 60-70% | Medium (handle interruptions) |
| Right-size node types | 20-40% | Low (gradual optimization) |
| Cluster Autoscaler | 15-30% | Low (automatic) |
| Use Fargate for burst workloads | Variable | Medium (per-pod pricing) |
| Reserved instances for base load | 30-40% | Low (1-3 year commitment) |

---

## Amazon MSK (Managed Streaming for Apache Kafka)

### Overview

Amazon MSK is a fully managed service for Apache Kafka, handling cluster setup, configuration, patching, and recovery.

### MSK Configuration

#### Production MSK Cluster

```hcl
resource "aws_msk_cluster" "production" {
  cluster_name           = "production-kafka-cluster"
  kafka_version          = "3.5.1"
  number_of_broker_nodes = 3

  broker_node_group_info {
    instance_type   = "kafka.m5.large"
    client_subnets  = module.vpc_production.data_subnets
    security_groups = [aws_security_group.msk_production.id]

    storage_info {
      ebs_storage_info {
        volume_size            = 500
        provisioned_throughput {
          enabled           = true
          volume_throughput = 250
        }
      }
    }

    connectivity_info {
      public_access {
        type = "DISABLED"
      }
    }
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"
      in_cluster    = true
    }
    encryption_at_rest_kms_key_arn = aws_kms_key.msk.arn
  }

  configuration_info {
    arn      = aws_msk_configuration.production.arn
    revision = aws_msk_configuration.production.latest_revision
  }

  client_authentication {
    sasl {
      iam   = true
      scram = false
    }
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.msk_broker_logs.name
      }
      s3 {
        enabled = true
        bucket  = aws_s3_bucket.msk_logs.id
        prefix  = "broker-logs/"
      }
    }
  }

  tags = {
    Environment = "production"
    Terraform   = "true"
  }
}

resource "aws_msk_configuration" "production" {
  name              = "production-kafka-config"
  kafka_versions    = ["3.5.1"]
  server_properties = <<PROPERTIES
auto.create.topics.enable = false
default.replication.factor = 3
min.insync.replicas = 2
num.io.threads = 8
num.network.threads = 5
num.replica.fetchers = 2
replica.lag.time.max.ms = 30000
socket.receive.buffer.bytes = 102400
socket.request.max.bytes = 104857600
socket.send.buffer.bytes = 102400
unclean.leader.election.enable = false
zookeeper.session.timeout.ms = 18000
log.retention.hours = 168
log.segment.bytes = 1073741824
compression.type = producer
PROPERTIES
}
```

#### Development MSK Cluster

```hcl
resource "aws_msk_cluster" "development" {
  cluster_name           = "dev-kafka-cluster"
  kafka_version          = "3.5.1"
  number_of_broker_nodes = 2  # Cost optimization

  broker_node_group_info {
    instance_type   = "kafka.t3.small"  # Smaller instance
    client_subnets  = module.vpc_dev.data_subnets
    security_groups = [aws_security_group.msk_dev.id]

    storage_info {
      ebs_storage_info {
        volume_size = 100  # Smaller storage
      }
    }
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS_PLAINTEXT"  # Allow plaintext for easier debugging
      in_cluster    = true
    }
  }

  tags = {
    Environment = "development"
    Terraform   = "true"
  }
}
```

### MSK Specifications

| Component | Development | Production |
|-----------|-------------|------------|
| **Cluster Configuration** |
| Kafka Version | 3.5.1 | 3.5.1 |
| Broker Nodes | 2 | 3 |
| Availability Zones | 2 | 3 |
| Instance Type | kafka.t3.small | kafka.m5.large |
| **Storage** |
| Volume Size per Broker | 100 GB | 500 GB |
| Volume Type | GP3 | GP3 |
| Provisioned Throughput | No | Yes (250 MB/s) |
| **Security** |
| Encryption In-Transit | TLS_PLAINTEXT | TLS |
| Encryption At-Rest | Default | KMS |
| Authentication | None | IAM |
| **Networking** |
| Public Access | No | No |
| Client Access | Within VPC | Within VPC |
| **Monitoring** |
| CloudWatch Logs | Basic | Enhanced |
| JMX Metrics | Enabled | Enabled |
| S3 Log Backup | No | Yes |
| **Configuration** |
| Auto-create Topics | true | false |
| Replication Factor | 2 | 3 |
| Min In-Sync Replicas | 1 | 2 |
| Log Retention | 24 hours | 7 days |

### MSK Features

#### 1. **High Availability**
- Multi-AZ deployment
- Automatic broker replacement
- Automatic partition rebalancing

#### 2. **Security**
- IAM authentication for clients
- TLS encryption for data in transit
- KMS encryption for data at rest
- VPC isolation

#### 3. **Monitoring**
- CloudWatch metrics integration
- JMX metrics export
- Broker log shipping to CloudWatch or S3

#### 4. **Scalability**
- Horizontal scaling (add brokers)
- Vertical scaling (change instance types)
- Storage expansion (increase EBS volumes)

### MSK Topic Strategy

```yaml
# Example topic configuration
topics:
  - name: orders.created
    partitions: 12
    replication: 3
    retention_hours: 168
    config:
      min.insync.replicas: 2
      cleanup.policy: delete
      
  - name: orders.updated
    partitions: 12
    replication: 3
    retention_hours: 168
    
  - name: users.events
    partitions: 6
    replication: 3
    retention_hours: 720  # 30 days
    
  - name: notifications
    partitions: 3
    replication: 3
    retention_hours: 24
    config:
      cleanup.policy: delete
      compression.type: gzip
```

### Cost Analysis

**Production MSK Cluster** (3 brokers, kafka.m5.large):
- Broker instances: 3 × $0.21/hour × 730 hours = ~$460/month
- Storage: 1,500 GB × $0.10/GB = ~$150/month
- Data transfer: ~$20-50/month
- **Total: ~$630-660/month**

**Development MSK Cluster** (2 brokers, kafka.t3.small):
- Broker instances: 2 × $0.042/hour × 730 hours = ~$61/month
- Storage: 200 GB × $0.10/GB = ~$20/month
- **Total: ~$81/month**

---

## Alternative: Self-Managed Kafka (Strimzi)

### Overview

Strimzi is a Kubernetes operator for running Apache Kafka on Kubernetes. It's an excellent alternative for learning, development, or when you need full control over Kafka configuration.

### Strimzi Architecture

```
Kubernetes Cluster
├── Strimzi Operator (watches Kafka CRDs)
├── Kafka Cluster
│   ├── Kafka Pods (3 replicas)
│   ├── ZooKeeper Pods (3 replicas) [or KRaft mode]
│   └── Entity Operator
│       ├── Topic Operator (manages topics)
│       └── User Operator (manages ACLs)
├── Kafka Connect Cluster (optional)
└── Kafka Bridge (optional, HTTP interface)
```

### Strimzi Installation

```yaml
# Install Strimzi operator
apiVersion: v1
kind: Namespace
metadata:
  name: kafka

---
# Install using Helm
# helm repo add strimzi https://strimzi.io/charts/
# helm install strimzi-operator strimzi/strimzi-kafka-operator -n kafka

# Or using kubectl
# kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka'
```

### Kafka Cluster Definition

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: production-kafka
  namespace: kafka
spec:
  kafka:
    version: 3.5.1
    replicas: 3
    
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
    
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      log.retention.hours: 168
      auto.create.topics.enable: false
      
    storage:
      type: persistent-claim
      size: 200Gi
      class: gp3
      deleteClaim: false
    
    resources:
      requests:
        memory: 2Gi
        cpu: "1"
      limits:
        memory: 4Gi
        cpu: "2"
    
    # Use spot instances
    template:
      pod:
        tolerations:
          - key: "spot"
            operator: "Equal"
            value: "true"
            effect: "NoSchedule"
        nodeSelector:
          workload-type: "flexible"
    
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: kafka-metrics-config.yml
  
  zookeeper:
    replicas: 3
    
    storage:
      type: persistent-claim
      size: 50Gi
      class: gp3
      deleteClaim: false
    
    resources:
      requests:
        memory: 1Gi
        cpu: "0.5"
      limits:
        memory: 2Gi
        cpu: "1"
    
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: zookeeper-metrics-config.yml
  
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

### Kafka Topic CRD

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders.created
  namespace: kafka
  labels:
    strimzi.io/cluster: production-kafka
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 604800000  # 7 days
    min.insync.replicas: 2
    cleanup.policy: delete
    compression.type: producer
```

### MSK vs Strimzi Comparison

| Feature | Amazon MSK | Strimzi on EKS |
|---------|-----------|----------------|
| **Management** |
| Setup Complexity | Low (AWS console/Terraform) | Medium (K8s operator) |
| Operational Overhead | Very Low (AWS managed) | Medium (you manage) |
| Upgrades | Automated by AWS | Manual (operator upgrade) |
| Patching | Automated by AWS | Manual |
| **Flexibility** |
| Kafka Configuration | Limited by AWS | Full control |
| Custom Plugins | Limited | Full support |
| Kafka Connect | Separate service | Integrated CRD |
| Kafka Versions | AWS-supported only | Any version |
| **Cost** |
| Production (3 brokers) | ~$630-660/month | ~$200-300/month* |
| Development (2 brokers) | ~$81/month | ~$50-80/month* |
| Spot Instance Support | No | Yes |
| **Performance** |
| Throughput | High (optimized by AWS) | High (tunable) |
| Latency | Low | Low |
| Storage | EBS (scalable) | EBS via PVC |
| **Availability** |
| Multi-AZ | Built-in | Configure via topology |
| Automatic Failover | Yes | Yes (K8s handles) |
| Disaster Recovery | AWS backup | Manual backup strategy |
| **Monitoring** |
| CloudWatch Integration | Native | Via exporters |
| Prometheus Metrics | Via JMX exporter | Native support |
| Logging | CloudWatch/S3 | EFK stack |
| **Security** |
| Encryption at Rest | KMS | K8s secrets + EBS |
| Encryption in Transit | TLS | TLS |
| Authentication | IAM, SASL | TLS, SCRAM, OAuth |
| Network Isolation | VPC | K8s Network Policies |

*Estimated based on EC2 spot instances + EBS costs

### Recommendation Matrix

| Scenario | Recommendation | Reason |
|----------|----------------|---------|
| Production, team < 5 | MSK | Less operational burden |
| Production, team > 5 with K8s expertise | Strimzi | Cost savings, flexibility |
| Production, compliance-heavy | MSK | AWS compliance certifications |
| Development | Strimzi | Cost savings, learning |
| Learning Kafka | Strimzi | Full control, visibility |
| Need custom Kafka plugins | Strimzi | Full flexibility |
| Need Kafka Connect | Strimzi | Integrated CRDs |
| Multi-cloud strategy | Strimzi | Portable across clouds |

---

## Networking Infrastructure

### VPC Design

Each environment has its own VPC with the following structure:

```
Production VPC: 10.0.0.0/16
Development VPC: 10.1.0.0/16
```

#### Subnet Layout

**Production VPC (10.0.0.0/16)**:

| Subnet Type | AZ | CIDR | Purpose |
|-------------|-----|------|---------|
| Public A | us-east-1a | 10.0.1.0/24 | ALB, NAT Gateway |
| Public B | us-east-1b | 10.0.2.0/24 | ALB, NAT Gateway |
| Public C | us-east-1c | 10.0.3.0/24 | ALB, NAT Gateway |
| Private A | us-east-1a | 10.0.11.0/24 | EKS Nodes |
| Private B | us-east-1b | 10.0.12.0/24 | EKS Nodes |
| Private C | us-east-1c | 10.0.13.0/24 | EKS Nodes |
| Control Plane A | us-east-1a | 10.0.21.0/24 | EKS Control Plane ENIs |
| Control Plane B | us-east-1b | 10.0.22.0/24 | EKS Control Plane ENIs |
| Control Plane C | us-east-1c | 10.0.23.0/24 | EKS Control Plane ENIs |
| Data A | us-east-1a | 10.0.31.0/24 | MSK, RDS |
| Data B | us-east-1b | 10.0.32.0/24 | MSK, RDS |
| Data C | us-east-1c | 10.0.33.0/24 | MSK, RDS |

**Development VPC (10.1.0.0/16)**: Same structure with 10.1.x.x addressing

### Terraform VPC Configuration

```hcl
module "vpc_production" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "production-vpc"
  cidr = "10.0.0.0/16"

  azs = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  # Public subnets (ALB, NAT Gateway)
  public_subnets = [
    "10.0.1.0/24",
    "10.0.2.0/24",
    "10.0.3.0/24"
  ]
  
  # Private subnets (EKS nodes)
  private_subnets = [
    "10.0.11.0/24",
    "10.0.12.0/24",
    "10.0.13.0/24"
  ]
  
  # Control plane subnets (EKS control plane)
  intra_subnets = [
    "10.0.21.0/24",
    "10.0.22.0/24",
    "10.0.23.0/24"
  ]
  
  # Data subnets (MSK, RDS)
  database_subnets = [
    "10.0.31.0/24",
    "10.0.32.0/24",
    "10.0.33.0/24"
  ]

  # NAT Gateway configuration
  enable_nat_gateway     = true
  single_nat_gateway     = false  # One NAT per AZ for HA
  one_nat_gateway_per_az = true

  # DNS
  enable_dns_hostnames = true
  enable_dns_support   = true

  # VPC Flow Logs
  enable_flow_log                      = true
  create_flow_log_cloudwatch_log_group = true
  create_flow_log_cloudwatch_iam_role  = true

  # Tags for EKS
  public_subnet_tags = {
    "kubernetes.io/role/elb"                    = 1
    "kubernetes.io/cluster/production-eks-cluster" = "shared"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb"           = 1
    "kubernetes.io/cluster/production-eks-cluster" = "shared"
  }

  tags = {
    Environment = "production"
    Terraform   = "true"
  }
}
```

### VPC Endpoints

To reduce NAT Gateway costs and improve security, create VPC endpoints for AWS services:

```hcl
# S3 Gateway Endpoint (no cost)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc_production.vpc_id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc_production.private_route_table_ids

  tags = {
    Name = "production-s3-endpoint"
  }
}

# ECR API Interface Endpoint
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = module.vpc_production.vpc_id
  service_name        = "com.amazonaws.us-east-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc_production.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "production-ecr-api-endpoint"
  }
}

# ECR Docker Interface Endpoint
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = module.vpc_production.vpc_id
  service_name        = "com.amazonaws.us-east-1.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc_production.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "production-ecr-dkr-endpoint"
  }
}

# Additional recommended endpoints:
# - ec2 (for EKS node registration)
# - sts (for IAM role assumption)
# - logs (for CloudWatch logging)
# - secretsmanager (for secrets access)
```

---

## Compute Resources

### EC2 Instance Types

#### EKS Worker Nodes

**Production - On-Demand Nodes (Critical Workloads)**:
- Instance Type: `t3.large` or `t3a.large`
- vCPU: 2
- Memory: 8 GB
- Network: Up to 5 Gbps
- Cost: ~$0.0832/hour (t3.large), ~$0.0752/hour (t3a.large)

**Production - Spot Nodes (Flexible Workloads)**:
- Instance Type: `t3.large`, `t3a.large`, `t2.large`
- Cost: ~$0.025-0.030/hour (70% discount)
- Interruption handling: 2-minute notice

**Development - Spot Nodes**:
- Instance Type: `t3.medium`
- vCPU: 2
- Memory: 4 GB
- Cost: ~$0.0125/hour (spot)

### Bastion Host

```hcl
resource "aws_instance" "bastion" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.micro"
  
  subnet_id                   = module.vpc_dev.public_subnets[0]
  vpc_security_group_ids      = [aws_security_group.bastion.id]
  associate_public_ip_address = true
  
  key_name = aws_key_pair.bastion.key_name
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y kubectl aws-cli jq git
              
              # Install kubectl
              curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
              chmod +x kubectl
              mv kubectl /usr/local/bin/
              
              # Install helm
              curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
              
              # Install k9s
              curl -sS https://webinstall.dev/k9s | bash
              EOF

  tags = {
    Name        = "dev-bastion-host"
    Environment = "development"
  }
}
```

Specifications:
- Instance Type: t3.micro
- Cost: ~$7.50/month
- Purpose: SSH access to private resources, kubectl access
- Availability: Dev environment only

---

## Amazon ECR (Elastic Container Registry)

### Repository Configuration

```hcl
resource "aws_ecr_repository" "app_repos" {
  for_each = toset([
    "api-gateway",
    "auth-service",
    "order-service",
    "user-service",
    "notification-service"
  ])

  name                 = each.key
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key        = aws_kms_key.ecr.arn
  }

  tags = {
    Service   = each.key
    ManagedBy = "terraform"
  }
}

# Lifecycle policy to clean up old images
resource "aws_ecr_lifecycle_policy" "app_repos" {
  for_each   = aws_ecr_repository.app_repos
  repository = each.value.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 10
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2
        description  = "Remove untagged images after 7 days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 7
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

### ECR Features

- **Image Scanning**: Automatic vulnerability scanning on push
- **Immutable Tags**: Prevent tag overwriting for production images
- **Lifecycle Policies**: Automatic cleanup of old images
- **Encryption**: KMS encryption at rest
- **Cross-Region Replication**: Optional for DR
- **IAM Integration**: Fine-grained access control

### Cost

- Storage: $0.10 per GB per month
- Data Transfer: First 50 TB/month out to Internet: $0.09/GB
- Typical costs: $5-20/month for small deployments

---

## Storage Solutions

### EBS (Elastic Block Store)

#### For EKS Persistent Volumes

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:ACCOUNT_ID:key/KEY_ID"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**gp3 Specifications**:
- Baseline: 3,000 IOPS, 125 MB/s throughput
- Cost: $0.08 per GB per month
- Max: 16,000 IOPS, 1,000 MB/s
- Use case: General purpose, databases

### S3 (Simple Storage Service)

#### Bucket Configuration

```hcl
# Application assets bucket
resource "aws_s3_bucket" "app_assets" {
  bucket = "production-app-assets-${data.aws_caller_identity.current.account_id}"

  tags = {
    Name        = "App Assets"
    Environment = "production"
  }
}

# Enable versioning
resource "aws_s3_bucket_versioning" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
  }
}

# Lifecycle rules
resource "aws_s3_bucket_lifecycle_configuration" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id

  rule {
    id     = "transition-to-ia"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }
  }
}

# Backup bucket for Kafka logs, database backups
resource "aws_s3_bucket" "backups" {
  bucket = "production-backups-${data.aws_caller_identity.current.account_id}"

  tags = {
    Name        = "Backups"
    Environment = "production"
  }
}
```

**S3 Cost Optimization**:
- Standard: $0.023/GB (first 50 TB)
- Standard-IA: $0.0125/GB (for infrequent access)
- Glacier Instant Retrieval: $0.004/GB (archival)
- Use lifecycle policies to move old data

---

## Database Services

### Amazon RDS/Aurora

#### Production: Aurora PostgreSQL

```hcl
module "aurora_production" {
  source  = "terraform-aws-modules/rds-aurora/aws"
  version = "~> 8.0"

  name           = "production-aurora-cluster"
  engine         = "aurora-postgresql"
  engine_version = "15.3"
  instance_class = "db.r6g.large"

  instances = {
    1 = {}  # Writer instance
    2 = {}  # Reader instance
  }

  vpc_id                 = module.vpc_production.vpc_id
  db_subnet_group_name   = module.vpc_production.database_subnet_group_name
  create_security_group  = true
  allowed_cidr_blocks    = module.vpc_production.private_subnets_cidr_blocks

  # Storage
  storage_encrypted = true
  kms_key_id       = aws_kms_key.rds.arn

  # Backups
  backup_retention_period      = 7
  preferred_backup_window      = "03:00-04:00"
  preferred_maintenance_window = "sun:04:00-sun:05:00"

  # Monitoring
  enabled_cloudwatch_logs_exports = ["postgresql"]
  monitoring_interval             = 60
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn

  # Database parameters
  database_name  = "production_db"
  master_username = "dbadmin"
  manage_master_user_password = true  # AWS Secrets Manager

  # High availability
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

  tags = {
    Environment = "production"
    Terraform   = "true"
  }
}
```

**Aurora Features**:
- Auto-scaling storage (up to 128 TB)
- Fast failover (<30 seconds)
- Up to 15 read replicas
- Continuous backup to S3
- Point-in-time recovery

**Cost (db.r6g.large, 2 instances)**:
- Writer: $0.29/hour × 730 hours = ~$212/month
- Reader: $0.29/hour × 730 hours = ~$212/month
- Storage: ~$0.10/GB-month
- I/O: $0.20 per 1 million requests
- **Total: ~$450-550/month**

#### Development: RDS PostgreSQL

```hcl
module "rds_development" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "dev-postgres"

  engine               = "postgres"
  engine_version       = "15.3"
  family               = "postgres15"
  major_engine_version = "15"
  instance_class       = "db.t3.micro"

  allocated_storage     = 20
  max_allocated_storage = 100
  storage_encrypted     = true

  db_name  = "dev_db"
  username = "dbadmin"
  port     = 5432

  multi_az               = false
  db_subnet_group_name   = module.vpc_dev.database_subnet_group_name
  vpc_security_group_ids = [aws_security_group.rds_dev.id]

  backup_retention_period = 1
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  tags = {
    Environment = "development"
  }
}
```

**Cost (db.t3.micro)**:
- Instance: $0.017/hour × 730 hours = ~$12.50/month
- Storage: 20 GB × $0.115/GB = ~$2.30/month
- **Total: ~$15/month**

---

## Load Balancing

### Application Load Balancer

```hcl
resource "aws_lb" "production" {
  name               = "production-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc_production.public_subnets

  enable_deletion_protection = true
  enable_http2              = true
  enable_cross_zone_load_balancing = true

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "production-alb"
    enabled = true
  }

  tags = {
    Environment = "production"
  }
}

# HTTPS listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.production.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.app.arn

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "Not Found"
      status_code  = "404"
    }
  }
}

# HTTP to HTTPS redirect
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.production.arn
  port              = "80"
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

### AWS Load Balancer Controller

```yaml
# Installed via Helm
# helm repo add eks https://aws.github.io/eks-charts
# helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
#   -n kube-system \
#   --set clusterName=production-eks-cluster \
#   --set serviceAccount.create=true \
#   --set serviceAccount.name=aws-load-balancer-controller

# Ingress example
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/CERT_ID
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 80
```

**Cost**:
- ALB: $0.0225/hour + $0.008/LCU-hour
- Typical small deployment: ~$20-30/month
- Medium traffic: ~$40-60/month

---

## DNS and Domain Management

### Route53

```hcl
# Hosted zone
resource "aws_route53_zone" "main" {
  name = "example.com"

  tags = {
    Environment = "production"
  }
}

# Production API
resource "aws_route53_record" "api_production" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.production.dns_name
    zone_id                = aws_lb.production.zone_id
    evaluate_target_health = true
  }
}

# Development API
resource "aws_route53_record" "api_dev" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api-dev.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.development.dns_name
    zone_id                = aws_lb.development.zone_id
    evaluate_target_health = true
  }
}

# Health check for production
resource "aws_route53_health_check" "api_production" {
  fqdn              = "api.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = "3"
  request_interval  = "30"

  tags = {
    Name = "api-production-health"
  }
}
```

**Cost**:
- Hosted zone: $0.50/month
- Queries: $0.40 per million queries (first 1B)
- Health checks: $0.50 each/month
- Typical cost: ~$2-5/month

---

## Infrastructure Comparison Tables

### Environment Comparison

| Component | Development | Production | Cost Ratio |
|-----------|-------------|------------|------------|
| EKS Control Plane | $72/month | $72/month | 1:1 |
| EKS Workers | ~$50/month (spot) | ~$600/month (mixed) | 1:12 |
| MSK/Kafka | ~$80/month | ~$650/month | 1:8 |
| RDS | ~$15/month | ~$500/month | 1:33 |
| NAT Gateway | $32/month (1 AZ) | $96/month (3 AZs) | 1:3 |
| Load Balancer | ~$20/month | ~$40/month | 1:2 |
| ECR/S3/Misc | ~$10/month | ~$40/month | 1:4 |
| **Total** | **~$279/month** | **~$1,998/month** | **1:7** |

### Scaling Timeline

| Metric | Initial | 6 Months | 1 Year | 2 Years |
|--------|---------|----------|--------|---------|
| EKS Nodes (Prod) | 5 | 10-15 | 15-25 | 25-40 |
| RDS Size | db.r6g.large | db.r6g.xlarge | db.r6g.2xlarge | Sharding |
| Kafka Brokers | 3 | 3 | 4-6 | 6-9 |
| Monthly Cost | ~$2,000 | ~$3,500 | ~$5,500 | ~$8,000+ |

---

## Summary

This infrastructure provides:

1. **Scalability**: From dev experiments to production scale
2. **High Availability**: Multi-AZ deployment across all critical components
3. **Security**: Encryption, private networks, IAM integration
4. **Cost Optimization**: Spot instances, right-sizing, lifecycle policies
5. **Operational Excellence**: Managed services reduce operational burden
6. **Flexibility**: Choice between managed (MSK) and self-managed (Strimzi) Kafka

**Key Decisions**:
- ✅ Separate VPCs per environment
- ✅ MSK for production (managed), Strimzi option for dev/learning
- ✅ Mixed on-demand/spot instances for cost optimization
- ✅ Aurora for production, standard RDS for dev
- ✅ Multi-AZ deployment for all production components

---

[← Back to Architecture Overview](./01-architecture-overview.md) | [Next: Network Architecture →](./03-network-architecture.md)
