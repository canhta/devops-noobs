# 08. Terraform Structure and Organization

## Table of Contents
1. [Overview](#overview)
2. [Repository Structure](#repository-structure)
3. [Module Organization](#module-organization)
4. [State Management](#state-management)
5. [Environment Configuration](#environment-configuration)
6. [Core Modules](#core-modules)
7. [Variables and Outputs](#variables-and-outputs)
8. [Best Practices](#best-practices)
9. [Workflow Integration](#workflow-integration)
10. [Complete Examples](#complete-examples)

---

## Overview

This document provides a complete Terraform infrastructure-as-code organization for the DevOps platform, following industry best practices for modularity, reusability, and maintainability.

### Design Principles

- **DRY (Don't Repeat Yourself)**: Reusable modules across environments
- **Environment Isolation**: Separate state files for dev and prod
- **Version Control**: Pin module and provider versions
- **Security**: Never commit secrets, use AWS Secrets Manager
- **Documentation**: Self-documenting code with clear variable descriptions
- **Testing**: Plan before apply, validate configurations

---

## Repository Structure

```
terraform/
├── modules/                          # Reusable Terraform modules
│   ├── networking/                   # VPC, subnets, NAT, etc.
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── eks/                         # EKS cluster configuration
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── irsa.tf                  # IAM Roles for Service Accounts
│   │   └── README.md
│   ├── msk/                         # MSK Kafka cluster
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── rds/                         # RDS/Aurora databases
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── ecr/                         # Container registries
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── security-groups/             # Security group definitions
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── iam/                         # IAM roles and policies
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── policies/
│   │       ├── eks-node-policy.json
│   │       └── github-actions-policy.json
│   ├── observability/               # Monitoring stack
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── helm-values/
│   │       ├── prometheus.yaml
│   │       ├── loki.yaml
│   │       └── grafana.yaml
│   └── bastion/                     # Bastion host
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── user-data.sh
│
├── environments/                     # Environment-specific configurations
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   ├── backend.tf
│   │   └── outputs.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       ├── backend.tf
│       └── outputs.tf
│
├── global/                          # Global/shared resources
│   ├── ecr/                        # ECR repositories (shared)
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── iam/                        # Cross-environment IAM
│   │   ├── github-oidc.tf
│   │   └── outputs.tf
│   └── route53/                    # DNS zones
│       ├── main.tf
│       └── outputs.tf
│
├── scripts/                        # Helper scripts
│   ├── init-backend.sh            # Initialize S3 backend
│   ├── plan-all.sh                # Plan all environments
│   ├── apply-env.sh               # Apply specific environment
│   └── destroy-env.sh             # Destroy environment (with confirmation)
│
└── README.md                       # Main documentation
```

---

## Module Organization

### Module Design Principles

Each module should:
1. **Single Responsibility**: Do one thing well
2. **Self-Contained**: Include all necessary resources
3. **Configurable**: Accept variables for customization
4. **Well-Documented**: Clear README with usage examples
5. **Tested**: Validate with `terraform validate` and `terraform plan`

### Module Template

```hcl
# modules/example/main.tf

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Module resources here
```

```hcl
# modules/example/variables.tf

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "tags" {
  description = "Common tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/example/outputs.tf

output "resource_id" {
  description = "The ID of the created resource"
  value       = aws_example_resource.this.id
}

output "resource_arn" {
  description = "The ARN of the created resource"
  value       = aws_example_resource.this.arn
}
```

---

## State Management

### S3 Backend Configuration

```hcl
# environments/dev/backend.tf

terraform {
  backend "s3" {
    bucket         = "terraform-state-ACCOUNT_ID"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    
    # Optional: Use role assumption for cross-account access
    # role_arn = "arn:aws:iam::ACCOUNT_ID:role/TerraformStateAccess"
  }
}
```

### Initialize Backend (one-time setup)

```bash
#!/bin/bash
# scripts/init-backend.sh

set -e

AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION="us-east-1"
BUCKET_NAME="terraform-state-${AWS_ACCOUNT_ID}"
DYNAMODB_TABLE="terraform-state-lock"

echo "Creating Terraform state S3 bucket: ${BUCKET_NAME}"

# Create S3 bucket
aws s3 mb "s3://${BUCKET_NAME}" --region "${AWS_REGION}" || echo "Bucket already exists"

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket "${BUCKET_NAME}" \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket "${BUCKET_NAME}" \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# Block public access
aws s3api put-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name "${DYNAMODB_TABLE}" \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region "${AWS_REGION}" || echo "DynamoDB table already exists"

echo "Terraform backend initialized successfully!"
echo "Bucket: ${BUCKET_NAME}"
echo "DynamoDB Table: ${DYNAMODB_TABLE}"
```

### State Management Best Practices

1. **Separate States**: Each environment has its own state file
2. **State Locking**: Use DynamoDB to prevent concurrent modifications
3. **Encryption**: Enable encryption at rest for state files
4. **Versioning**: Enable S3 versioning for state file recovery
5. **Access Control**: Restrict access to state bucket using IAM
6. **No Secrets**: Never store secrets in state (use AWS Secrets Manager)

---

## Environment Configuration

### Development Environment

```hcl
# environments/dev/main.tf

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Project     = "DevOps Platform"
    }
  }
}

locals {
  environment = "dev"
  aws_region  = "us-east-1"
  
  common_tags = {
    Environment = local.environment
    ManagedBy   = "Terraform"
    Project     = "DevOps Platform"
  }
}

# Networking
module "vpc" {
  source = "../../modules/networking"
  
  environment         = local.environment
  vpc_cidr           = var.vpc_cidr
  availability_zones = var.availability_zones
  
  # Cost optimization: single NAT gateway for dev
  enable_nat_gateway     = true
  single_nat_gateway     = true
  one_nat_gateway_per_az = false
  
  tags = local.common_tags
}

# EKS Cluster
module "eks" {
  source = "../../modules/eks"
  
  environment        = local.environment
  cluster_name       = "${local.environment}-eks-cluster"
  cluster_version    = "1.28"
  
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
  control_plane_subnet_ids = module.vpc.control_plane_subnet_ids
  
  # Smaller nodes for dev
  node_groups = {
    general = {
      instance_types = ["t3.medium"]
      capacity_type  = "SPOT"
      min_size       = 1
      max_size       = 5
      desired_size   = 2
    }
  }
  
  tags = local.common_tags
}

# MSK Kafka (smaller for dev)
module "msk" {
  source = "../../modules/msk"
  
  environment             = local.environment
  cluster_name            = "${local.environment}-kafka-cluster"
  kafka_version           = "3.5.1"
  number_of_broker_nodes  = 2  # Cost optimization
  broker_instance_type    = "kafka.t3.small"
  broker_volume_size      = 100
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.data_subnet_ids
  
  tags = local.common_tags
}

# RDS PostgreSQL
module "rds" {
  source = "../../modules/rds"
  
  environment = local.environment
  identifier  = "${local.environment}-postgres"
  
  engine         = "postgres"
  engine_version = "15.3"
  instance_class = "db.t3.micro"
  
  allocated_storage = 20
  multi_az         = false  # Cost optimization for dev
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.data_subnet_ids
  
  allowed_cidr_blocks = module.vpc.private_subnet_cidrs
  
  tags = local.common_tags
}

# Bastion Host
module "bastion" {
  source = "../../modules/bastion"
  
  environment         = local.environment
  vpc_id              = module.vpc.vpc_id
  public_subnet_id    = module.vpc.public_subnet_ids[0]
  
  allowed_cidr_blocks = var.bastion_allowed_cidrs
  
  tags = local.common_tags
}

# Observability Stack
module "observability" {
  source = "../../modules/observability"
  
  environment  = local.environment
  cluster_name = module.eks.cluster_name
  cluster_endpoint = module.eks.cluster_endpoint
  cluster_ca_certificate = module.eks.cluster_ca_certificate
  
  enable_prometheus = true
  enable_loki       = true
  enable_grafana    = true
  
  tags = local.common_tags
}
```

```hcl
# environments/dev/variables.tf

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.1.0.0/16"
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

variable "bastion_allowed_cidrs" {
  description = "CIDR blocks allowed to SSH to bastion"
  type        = list(string)
  # Should be overridden in terraform.tfvars
}
```

```hcl
# environments/dev/terraform.tfvars

aws_region         = "us-east-1"
environment        = "dev"
vpc_cidr           = "10.1.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b"]

# Override with your office/home IP
bastion_allowed_cidrs = ["203.0.113.0/24"]  # Example - replace with your IP
```

```hcl
# environments/dev/outputs.tf

output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "eks_cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
  sensitive   = true
}

output "eks_cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "msk_bootstrap_brokers" {
  description = "MSK Kafka bootstrap brokers"
  value       = module.msk.bootstrap_brokers
  sensitive   = true
}

output "rds_endpoint" {
  description = "RDS endpoint"
  value       = module.rds.endpoint
  sensitive   = true
}

output "bastion_public_ip" {
  description = "Bastion host public IP"
  value       = module.bastion.public_ip
}

# Generate kubeconfig command
output "configure_kubectl" {
  description = "Command to configure kubectl"
  value       = "aws eks update-kubeconfig --region ${var.aws_region} --name ${module.eks.cluster_name}"
}
```

### Production Environment

Production follows the same structure but with production-appropriate settings:

```hcl
# environments/prod/terraform.tfvars

aws_region         = "us-east-1"
environment        = "prod"
vpc_cidr           = "10.0.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

# Production uses more robust configuration:
# - 3 AZs instead of 2
# - Multi-AZ RDS
# - Larger instance types
# - More broker nodes
# - NAT Gateway per AZ
```

---

## Core Modules

### Networking Module

```hcl
# modules/networking/main.tf

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(
    var.tags,
    {
      Name = "${var.environment}-vpc"
    }
  )
}

# Public Subnets (for ALB, NAT)
resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index + 1)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(
    var.tags,
    {
      Name                                        = "${var.environment}-public-${var.availability_zones[count.index]}"
      "kubernetes.io/role/elb"                    = "1"
      "kubernetes.io/cluster/${var.environment}-eks-cluster" = "shared"
    }
  )
}

# Private Subnets (for EKS nodes)
resource "aws_subnet" "private" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 11)
  availability_zone = var.availability_zones[count.index]
  
  tags = merge(
    var.tags,
    {
      Name                                        = "${var.environment}-private-${var.availability_zones[count.index]}"
      "kubernetes.io/role/internal-elb"           = "1"
      "kubernetes.io/cluster/${var.environment}-eks-cluster" = "shared"
    }
  )
}

# Control Plane Subnets (for EKS control plane ENIs)
resource "aws_subnet" "control_plane" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 21)
  availability_zone = var.availability_zones[count.index]
  
  tags = merge(
    var.tags,
    {
      Name = "${var.environment}-control-plane-${var.availability_zones[count.index]}"
    }
  )
}

# Data Subnets (for MSK, RDS)
resource "aws_subnet" "data" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 31)
  availability_zone = var.availability_zones[count.index]
  
  tags = merge(
    var.tags,
    {
      Name = "${var.environment}-data-${var.availability_zones[count.index]}"
    }
  )
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.environment}-igw"
    }
  )
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count  = var.single_nat_gateway ? 1 : (var.one_nat_gateway_per_az ? length(var.availability_zones) : 1)
  domain = "vpc"
  
  tags = merge(
    var.tags,
    {
      Name = "${var.environment}-nat-eip-${count.index + 1}"
    }
  )
  
  depends_on = [aws_internet_gateway.main]
}

# NAT Gateways
resource "aws_nat_gateway" "main" {
  count = var.single_nat_gateway ? 1 : (var.one_nat_gateway_per_az ? length(var.availability_zones) : 1)
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.environment}-nat-${count.index + 1}"
    }
  )
  
  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(
    var.tags,
    {
      Name = "${var.environment}-public-rt"
    }
  )
}

resource "aws_route_table_association" "public" {
  count = length(var.availability_zones)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  count  = var.single_nat_gateway ? 1 : length(var.availability_zones)
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = var.single_nat_gateway ? aws_nat_gateway.main[0].id : aws_nat_gateway.main[count.index].id
  }
  
  tags = merge(
    var.tags,
    {
      Name = "${var.environment}-private-rt-${count.index + 1}"
    }
  )
}

resource "aws_route_table_association" "private" {
  count = length(var.availability_zones)
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = var.single_nat_gateway ? aws_route_table.private[0].id : aws_route_table.private[count.index].id
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  count = var.enable_flow_logs ? 1 : 0
  
  iam_role_arn    = aws_iam_role.flow_logs[0].arn
  log_destination = aws_cloudwatch_log_group.flow_logs[0].arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.environment}-vpc-flow-logs"
    }
  )
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0
  
  name              = "/aws/vpc/${var.environment}-flow-logs"
  retention_in_days = 7
  
  tags = var.tags
}

resource "aws_iam_role" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0
  
  name = "${var.environment}-vpc-flow-logs-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
      }
    ]
  })
  
  tags = var.tags
}

resource "aws_iam_role_policy" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0
  
  name = "${var.environment}-vpc-flow-logs-policy"
  role = aws_iam_role.flow_logs[0].id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}
```

```hcl
# modules/networking/outputs.tf

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "private_subnet_cidrs" {
  description = "List of private subnet CIDR blocks"
  value       = aws_subnet.private[*].cidr_block
}

output "control_plane_subnet_ids" {
  description = "List of control plane subnet IDs"
  value       = aws_subnet.control_plane[*].id
}

output "data_subnet_ids" {
  description = "List of data subnet IDs"
  value       = aws_subnet.data[*].id
}

output "nat_gateway_ips" {
  description = "List of NAT Gateway public IPs"
  value       = aws_eip.nat[*].public_ip
}
```

### EKS Module

```hcl
# modules/eks/main.tf

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = var.vpc_id
  subnet_ids = var.private_subnet_ids
  control_plane_subnet_ids = var.control_plane_subnet_ids

  cluster_endpoint_public_access  = var.cluster_endpoint_public_access
  cluster_endpoint_private_access = var.cluster_endpoint_private_access

  # Enable IRSA (IAM Roles for Service Accounts)
  enable_irsa = true

  # Cluster encryption
  cluster_encryption_config = var.enable_secrets_encryption ? {
    resources        = ["secrets"]
    provider_key_arn = aws_kms_key.eks[0].arn
  } : {}

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
      most_recent              = true
      service_account_role_arn = aws_iam_role.ebs_csi_driver.arn
    }
  }

  # Node groups
  eks_managed_node_groups = var.node_groups

  # Cluster security group rules
  cluster_security_group_additional_rules = {
    ingress_nodes_ephemeral_ports_tcp = {
      description                = "Nodes on ephemeral ports"
      protocol                   = "tcp"
      from_port                  = 1025
      to_port                    = 65535
      type                       = "ingress"
      source_node_security_group = true
    }
  }

  # Node security group rules
  node_security_group_additional_rules = {
    ingress_self_all = {
      description = "Node to node all traffic"
      protocol    = "-1"
      from_port   = 0
      to_port     = 0
      type        = "ingress"
      self        = true
    }
    
    ingress_cluster_all = {
      description                   = "Cluster to node all traffic"
      protocol                      = "-1"
      from_port                     = 0
      to_port                       = 0
      type                          = "ingress"
      source_cluster_security_group = true
    }
    
    egress_all = {
      description      = "Node all egress"
      protocol         = "-1"
      from_port        = 0
      to_port          = 0
      type             = "egress"
      cidr_blocks      = ["0.0.0.0/0"]
      ipv6_cidr_blocks = ["::/0"]
    }
  }

  # CloudWatch Logging
  cluster_enabled_log_types = var.cluster_enabled_log_types

  tags = var.tags
}

# KMS key for EKS secrets encryption
resource "aws_kms_key" "eks" {
  count = var.enable_secrets_encryption ? 1 : 0
  
  description             = "KMS key for EKS ${var.cluster_name} secrets encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  tags = merge(
    var.tags,
    {
      Name = "${var.cluster_name}-eks-secrets-key"
    }
  )
}

resource "aws_kms_alias" "eks" {
  count = var.enable_secrets_encryption ? 1 : 0
  
  name          = "alias/${var.cluster_name}-eks-secrets"
  target_key_id = aws_kms_key.eks[0].key_id
}

# IAM role for EBS CSI driver
resource "aws_iam_role" "ebs_csi_driver" {
  name = "${var.cluster_name}-ebs-csi-driver"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRoleWithWebIdentity"
        Effect = "Allow"
        Principal = {
          Federated = module.eks.oidc_provider_arn
        }
        Condition = {
          StringEquals = {
            "${replace(module.eks.oidc_provider, "https://", "")}:sub" = "system:serviceaccount:kube-system:ebs-csi-controller-sa"
            "${replace(module.eks.oidc_provider, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "ebs_csi_driver" {
  role       = aws_iam_role.ebs_csi_driver.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
}
```

---

## Variables and Outputs

### Common Variable Patterns

```hcl
# Standard environment variable
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Tags variable (used in all modules)
variable "tags" {
  description = "Common tags to apply to all resources"
  type        = map(string)
  default     = {}
}

# Feature flag
variable "enable_feature" {
  description = "Enable optional feature"
  type        = bool
  default     = false
}

# List of strings
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  
  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones are required."
  }
}

# Complex object
variable "node_groups" {
  description = "EKS node group configurations"
  type = map(object({
    instance_types = list(string)
    capacity_type  = string
    min_size       = number
    max_size       = number
    desired_size   = number
  }))
  default = {}
}
```

---

## Best Practices

### 1. **Version Pinning**

```hcl
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Pin to major version
    }
  }
}
```

### 2. **Default Tags**

```hcl
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Project     = "DevOps Platform"
      CostCenter  = "Engineering"
    }
  }
}
```

### 3. **Naming Conventions**

```
Resource: {environment}-{service}-{resource-type}
Examples:
- prod-eks-cluster
- dev-kafka-cluster
- prod-api-alb
```

### 4. **Sensitive Data**

```hcl
output "database_password" {
  description = "Database password"
  value       = aws_db_instance.main.password
  sensitive   = true  # Hide from logs
}
```

### 5. **Resource Dependencies**

```hcl
# Explicit dependency
resource "aws_instance" "example" {
  # ...
  
  depends_on = [aws_iam_role_policy.example]
}

# Implicit dependency (preferred when possible)
resource "aws_instance" "example" {
  subnet_id = aws_subnet.example.id  # Implicit dependency
}
```

---

## Workflow Integration

### Development Workflow

```bash
# 1. Initialize
cd terraform/environments/dev
terraform init

# 2. Plan
terraform plan -out=tfplan

# 3. Review plan
terraform show tfplan

# 4. Apply
terraform apply tfplan

# 5. Output values
terraform output
```

### Script Examples

```bash
#!/bin/bash
# scripts/plan-all.sh

set -e

ENVIRONMENTS=("dev" "prod")

for env in "${ENVIRONMENTS[@]}"; do
  echo "Planning $env environment..."
  cd "terraform/environments/$env"
  terraform init -upgrade
  terraform plan -out="tfplan-$env"
  cd -
done

echo "All plans completed!"
```

---

## Complete Examples

### Global ECR Module

```hcl
# global/ecr/main.tf

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "terraform-state-ACCOUNT_ID"
    key            = "global/ecr/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = "us-east-1"
}

locals {
  repositories = [
    "api-gateway",
    "auth-service",
    "order-service",
    "user-service",
    "notification-service"
  ]
}

resource "aws_ecr_repository" "repos" {
  for_each = toset(local.repositories)

  name                 = each.key
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = {
    Service   = each.key
    ManagedBy = "Terraform"
  }
}

resource "aws_ecr_lifecycle_policy" "repos" {
  for_each   = aws_ecr_repository.repos
  repository = each.value.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 tagged images"
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

output "repository_urls" {
  description = "ECR repository URLs"
  value       = { for k, v in aws_ecr_repository.repos : k => v.repository_url }
}
```

---

## Summary

This Terraform structure provides:

✅ **Modularity**: Reusable modules across environments
✅ **Separation**: Clear environment isolation
✅ **State Management**: Secure remote state with locking
✅ **Best Practices**: Version pinning, tagging, naming conventions
✅ **Automation**: CI/CD integration with GitHub Actions
✅ **Documentation**: Self-documenting code
✅ **Scalability**: Easy to add new environments or services

### Key Features

- **DRY**: Modules prevent code duplication
- **Secure**: OIDC authentication, encrypted state
- **Flexible**: Easy to customize per environment
- **Production-Ready**: Follows AWS and Terraform best practices

---

[← Back to Kafka Architecture](./07-kafka-architecture.md) | [Next: Kubernetes Manifests →](./09-kubernetes-manifests.md)
