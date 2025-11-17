# 03. Network Architecture

## Table of Contents
1. [Overview](#overview)
2. [VPC Design](#vpc-design)
3. [Subnet Strategy](#subnet-strategy)
4. [Routing and NAT](#routing-and-nat)
5. [Security Groups](#security-groups)
6. [Network ACLs](#network-acls)
7. [VPC Endpoints](#vpc-endpoints)
8. [DNS and Route53](#dns-and-route53)
9. [Network Flow](#network-flow)
10. [Security Considerations](#security-considerations)

---

## Overview

This document provides detailed network architecture design for the DevOps platform, covering VPC configuration, subnet layout, routing, security groups, and network security best practices.

### Network Design Principles

- **Isolation**: Separate environments (dev/prod) with dedicated VPCs
- **Multi-AZ**: High availability across multiple availability zones
- **Security Layers**: Defense in depth with security groups and NACLs
- **Private by Default**: Workloads in private subnets, minimal public exposure
- **Controlled Egress**: NAT Gateways for outbound internet access
- **Service Connectivity**: VPC endpoints to reduce NAT costs and improve security

---

## VPC Design

### VPC Configuration

**Production VPC**:
```
CIDR Block: 10.0.0.0/16
Region: us-east-1
Availability Zones: us-east-1a, us-east-1b, us-east-1c
Total IP Addresses: 65,536
DNS Hostnames: Enabled
DNS Resolution: Enabled
```

**Development VPC**:
```
CIDR Block: 10.1.0.0/16
Region: us-east-1
Availability Zones: us-east-1a, us-east-1b
Total IP Addresses: 65,536
DNS Hostnames: Enabled
DNS Resolution: Enabled
```

### VPC Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Production VPC (10.0.0.0/16)                 │
│                                                                       │
│  ┌────────────────────┬────────────────────┬────────────────────┐  │
│  │   AZ us-east-1a    │   AZ us-east-1b    │   AZ us-east-1c    │  │
│  │                    │                    │                    │  │
│  │  Public Subnet     │  Public Subnet     │  Public Subnet     │  │
│  │  10.0.1.0/24       │  10.0.2.0/24       │  10.0.3.0/24       │  │
│  │  ┌──────────────┐  │  ┌──────────────┐  │  ┌──────────────┐  │  │
│  │  │ NAT Gateway  │  │  │ NAT Gateway  │  │  │ NAT Gateway  │  │  │
│  │  │ ALB          │  │  │ ALB          │  │  │ ALB          │  │  │
│  │  └──────────────┘  │  └──────────────┘  │  └──────────────┘  │  │
│  │         │          │         │          │         │          │  │
│  ├─────────┼──────────┼─────────┼──────────┼─────────┼──────────┤  │
│  │         ▼          │         ▼          │         ▼          │  │
│  │  Private Subnet    │  Private Subnet    │  Private Subnet    │  │
│  │  10.0.11.0/24      │  10.0.12.0/24      │  10.0.13.0/24      │  │
│  │  ┌──────────────┐  │  ┌──────────────┐  │  ┌──────────────┐  │  │
│  │  │ EKS Nodes    │  │  │ EKS Nodes    │  │  │ EKS Nodes    │  │  │
│  │  │ Pods         │  │  │ Pods         │  │  │ Pods         │  │  │
│  │  └──────────────┘  │  └──────────────┘  │  └──────────────┘  │  │
│  │                    │                    │                    │  │
│  │  Control Plane     │  Control Plane     │  Control Plane     │  │
│  │  10.0.21.0/24      │  10.0.22.0/24      │  10.0.23.0/24      │  │
│  │  (EKS ENIs)        │  (EKS ENIs)        │  (EKS ENIs)        │  │
│  │                    │                    │                    │  │
│  │  Data Subnet       │  Data Subnet       │  Data Subnet       │  │
│  │  10.0.31.0/24      │  10.0.32.0/24      │  10.0.33.0/24      │  │
│  │  ┌──────────────┐  │  ┌──────────────┐  │  ┌──────────────┐  │  │
│  │  │ MSK Kafka    │  │  │ MSK Kafka    │  │  │ MSK Kafka    │  │  │
│  │  │ RDS Aurora   │  │  │ RDS Aurora   │  │  │ RDS Aurora   │  │  │
│  │  └──────────────┘  │  └──────────────┘  │  └──────────────┘  │  │
│  └────────────────────┴────────────────────┴────────────────────┘  │
│                                                                       │
│  Internet Gateway: igw-prod                                          │
│  NAT Gateways: 3 (one per AZ)                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Subnet Strategy

### Subnet Types and Purpose

The VPC is divided into four distinct subnet types, each serving specific purposes:

#### 1. Public Subnets
- **Purpose**: Resources requiring direct internet access
- **Use Cases**: NAT Gateways, Application Load Balancers, Bastion hosts
- **Internet Access**: Direct via Internet Gateway
- **CIDR Ranges**:
  - Production: 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24
  - Development: 10.1.1.0/24, 10.1.2.0/24

#### 2. Private Subnets
- **Purpose**: Application workloads (EKS nodes and pods)
- **Use Cases**: Kubernetes worker nodes, microservices
- **Internet Access**: Outbound via NAT Gateway
- **CIDR Ranges**:
  - Production: 10.0.11.0/24, 10.0.12.0/24, 10.0.13.0/24
  - Development: 10.1.11.0/24, 10.1.12.0/24

#### 3. Control Plane Subnets
- **Purpose**: EKS control plane ENIs
- **Use Cases**: EKS API server communication
- **Internet Access**: Managed by AWS
- **CIDR Ranges**:
  - Production: 10.0.21.0/24, 10.0.22.0/24, 10.0.23.0/24
  - Development: 10.1.21.0/24, 10.1.22.0/24

#### 4. Data Subnets
- **Purpose**: Stateful data services
- **Use Cases**: RDS, Aurora, MSK Kafka
- **Internet Access**: None (fully private)
- **CIDR Ranges**:
  - Production: 10.0.31.0/24, 10.0.32.0/24, 10.0.33.0/24
  - Development: 10.1.31.0/24, 10.1.32.0/24

### Terraform Subnet Configuration

```hcl
# modules/networking/subnets.tf
resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index + 1)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name                                        = "${var.environment}-public-${var.availability_zones[count.index]}"
    "kubernetes.io/role/elb"                    = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 11)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name                                        = "${var.environment}-private-${var.availability_zones[count.index]}"
    "kubernetes.io/role/internal-elb"           = "1"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  }
}

resource "aws_subnet" "data" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 31)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.environment}-data-${var.availability_zones[count.index]}"
  }
}
```

---

## Routing and NAT

### Route Tables

Each subnet type has its own route table with specific routing rules.

#### Public Route Table
```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.environment}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

#### Private Route Tables (per AZ)
```hcl
resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.environment}-private-rt-${var.availability_zones[count.index]}"
  }
}
```

### NAT Gateway Configuration

**Production**: 3 NAT Gateways (one per AZ) for high availability
**Development**: 1 NAT Gateway (cost optimization)

```hcl
# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count  = var.environment == "prod" ? length(var.availability_zones) : 1
  domain = "vpc"

  tags = {
    Name = "${var.environment}-nat-eip-${count.index + 1}"
  }
}

# NAT Gateways in public subnets
resource "aws_nat_gateway" "main" {
  count         = var.environment == "prod" ? length(var.availability_zones) : 1
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.environment}-nat-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}
```

---

## Security Groups

### Security Group Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Security Groups                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ALB Security Group         → EKS Node Security Group        │
│  ┌──────────────────┐          ┌──────────────────┐         │
│  │ Inbound:         │          │ Inbound:         │         │
│  │ - 80 (HTTP)      │─────────▶│ - 30000-32767    │         │
│  │ - 443 (HTTPS)    │          │   from ALB SG    │         │
│  │   from 0.0.0.0/0 │          │ - All traffic    │         │
│  │                  │          │   from Node SG   │         │
│  │ Outbound: All    │          │                  │         │
│  └──────────────────┘          │ Outbound: All    │         │
│                                 └──────────────────┘         │
│                                          │                    │
│  EKS Control Plane SG                    │                    │
│  ┌──────────────────┐                    │                    │
│  │ Inbound:         │                    │                    │
│  │ - 443 from       │◀───────────────────┘                    │
│  │   Node SG        │                                         │
│  │                  │          RDS Security Group             │
│  │ Outbound:        │          ┌──────────────────┐         │
│  │ - All to Node SG │          │ Inbound:         │         │
│  └──────────────────┘          │ - 5432 (PG)      │         │
│                                 │   from Node SG   │         │
│                                 │                  │         │
│  MSK Security Group             │ Outbound: All    │         │
│  ┌──────────────────┐          └──────────────────┘         │
│  │ Inbound:         │                                         │
│  │ - 9092, 9094     │                                         │
│  │   from Node SG   │                                         │
│  │                  │                                         │
│  │ Outbound: All    │                                         │
│  └──────────────────┘                                         │
└──────────────────────────────────────────────────────────────┘
```

### Terraform Security Group Definitions

```hcl
# modules/networking/security_groups.tf

# ALB Security Group
resource "aws_security_group" "alb" {
  name_prefix = "${var.environment}-alb-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for Application Load Balancer"

  ingress {
    description = "HTTP from internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.environment}-alb-sg"
  }
}

# EKS Node Security Group
resource "aws_security_group" "eks_nodes" {
  name_prefix = "${var.environment}-eks-nodes-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for EKS worker nodes"

  ingress {
    description     = "Allow pods to communicate with each other"
    from_port       = 0
    to_port         = 65535
    protocol        = "tcp"
    self            = true
  }

  ingress {
    description     = "Allow ALB to reach NodePort services"
    from_port       = 30000
    to_port         = 32767
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name                                        = "${var.environment}-eks-nodes-sg"
    "kubernetes.io/cluster/${var.cluster_name}" = "owned"
  }
}

# RDS Security Group
resource "aws_security_group" "rds" {
  name_prefix = "${var.environment}-rds-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for RDS PostgreSQL"

  ingress {
    description     = "PostgreSQL from EKS nodes"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.environment}-rds-sg"
  }
}

# MSK Security Group
resource "aws_security_group" "msk" {
  name_prefix = "${var.environment}-msk-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for Amazon MSK"

  ingress {
    description     = "Kafka plaintext from EKS"
    from_port       = 9092
    to_port         = 9092
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }

  ingress {
    description     = "Kafka TLS from EKS"
    from_port       = 9094
    to_port         = 9094
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }

  ingress {
    description = "Zookeeper"
    from_port   = 2181
    to_port     = 2181
    protocol    = "tcp"
    self        = true
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.environment}-msk-sg"
  }
}
```

---

## Network ACLs

### NACL Strategy

Network ACLs provide an additional layer of security at the subnet level. We use them for:
- Blocking known malicious IPs
- Restricting protocols at network boundary
- Compliance requirements

```hcl
# Public Subnet NACL
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id

  # Allow inbound HTTP/HTTPS
  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  ingress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  # Allow return traffic
  ingress {
    protocol   = "tcp"
    rule_no    = 120
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  # Allow all outbound
  egress {
    protocol   = "-1"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = {
    Name = "${var.environment}-public-nacl"
  }
}
```

---

## VPC Endpoints

### Cost Optimization with VPC Endpoints

VPC endpoints reduce NAT Gateway costs by providing private connections to AWS services.

**Recommended Endpoints**:
- **S3 (Gateway)**: Free, essential for ECR image pulls
- **ECR API & Docker (Interface)**: ~$7.50/month, reduces data transfer costs
- **CloudWatch Logs (Interface)**: ~$7.50/month, improves log delivery
- **Secrets Manager (Interface)**: ~$7.50/month, secure secret retrieval

```hcl
# S3 Gateway Endpoint (Free)
resource "aws_vpc_endpoint" "s3" {
  vpc_id          = aws_vpc.main.id
  service_name    = "com.amazonaws.${var.region}.s3"
  route_table_ids = concat(
    [aws_route_table.public.id],
    aws_route_table.private[*].id
  )

  tags = {
    Name = "${var.environment}-s3-endpoint"
  }
}

# ECR API Endpoint
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.environment}-ecr-api-endpoint"
  }
}

# ECR Docker Endpoint
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.environment}-ecr-dkr-endpoint"
  }
}

# VPC Endpoint Security Group
resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "${var.environment}-vpce-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for VPC endpoints"

  ingress {
    description     = "HTTPS from VPC"
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    cidr_blocks     = [var.vpc_cidr]
  }

  tags = {
    Name = "${var.environment}-vpce-sg"
  }
}
```

---

## DNS and Route53

### DNS Configuration

**Private Hosted Zone** for internal service discovery:

```hcl
resource "aws_route53_zone" "private" {
  name = "${var.environment}.internal"

  vpc {
    vpc_id = aws_vpc.main.id
  }

  tags = {
    Name = "${var.environment}-private-zone"
  }
}

# Example: RDS endpoint record
resource "aws_route53_record" "rds" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "database.${var.environment}.internal"
  type    = "CNAME"
  ttl     = 300
  records = [aws_db_instance.main.address]
}

# Example: Kafka endpoint record
resource "aws_route53_record" "kafka" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "kafka.${var.environment}.internal"
  type    = "CNAME"
  ttl     = 300
  records = [aws_msk_cluster.main.bootstrap_brokers]
}
```

---

## Network Flow

### Inbound Traffic Flow

```
Internet → ALB (Public Subnet) → Target Group → 
EKS NodePort Service (Private Subnet) → Pod
```

### Outbound Traffic Flow

```
Pod → Node → NAT Gateway (Public Subnet) → Internet Gateway → Internet
```

### Inter-Service Communication

```
Pod A → Kubernetes Service → Pod B (same cluster, direct)
Pod → RDS/MSK (via Security Group rules, stays in VPC)
```

---

## Security Considerations

### Best Practices

1. **Principle of Least Privilege**
   - Only open required ports in security groups
   - Use specific CIDR blocks instead of 0.0.0.0/0 where possible

2. **Defense in Depth**
   - Security Groups + NACLs
   - WAF on ALB for production
   - VPC Flow Logs enabled

3. **Network Segmentation**
   - Separate subnets for different tiers
   - Data layer completely isolated
   - Control plane in dedicated subnets

4. **Monitoring**
   - Enable VPC Flow Logs
   - CloudWatch alarms for unusual traffic patterns
   - Regular security group audits

### VPC Flow Logs

```hcl
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn

  tags = {
    Name = "${var.environment}-vpc-flow-logs"
  }
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/${var.environment}"
  retention_in_days = 7

  tags = {
    Name = "${var.environment}-vpc-flow-logs"
  }
}
```

---

[← Back to Infrastructure Components](./02-infrastructure-components.md) | [Next: Security Model →](./04-security-model.md)
