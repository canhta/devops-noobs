# 20. Step-by-Step Implementation Guide

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Phase 1: Foundation Setup](#phase-1-foundation-setup)
4. [Phase 2: Infrastructure Provisioning](#phase-2-infrastructure-provisioning)
5. [Phase 3: Application Deployment](#phase-3-application-deployment)
6. [Phase 4: CI/CD Configuration](#phase-4-cicd-configuration)
7. [Phase 5: Observability Setup](#phase-5-observability-setup)
8. [Phase 6: Production Deployment](#phase-6-production-deployment)
9. [Phase 7: Operations and Maintenance](#phase-7-operations-and-maintenance)
10. [Troubleshooting](#troubleshooting)
11. [Validation Checklist](#validation-checklist)

---

## Overview

This guide provides a complete, beginner-friendly walkthrough for implementing the entire DevOps infrastructure from scratch. Follow each step in order for the best results.

### Time Estimates

| Phase | Estimated Time | Skill Level |
|-------|---------------|-------------|
| Phase 1: Prerequisites | 2-4 hours | Beginner |
| Phase 2: Infrastructure | 4-6 hours | Intermediate |
| Phase 3: Applications | 3-4 hours | Intermediate |
| Phase 4: CI/CD | 3-4 hours | Intermediate |
| Phase 5: Observability | 2-3 hours | Intermediate |
| Phase 6: Production | 2-3 hours | Advanced |
| **Total** | **16-24 hours** | **Split over 3-5 days** |

### Learning Approach

- **Day 1**: Phases 1-2 (Setup + Infrastructure)
- **Day 2**: Phase 3 (Application Deployment)
- **Day 3**: Phases 4-5 (CI/CD + Monitoring)
- **Day 4**: Phase 6 (Production)
- **Day 5**: Testing, optimization, and documentation

---

## Prerequisites

### 1. AWS Account Setup

**Create AWS Account**:
```bash
# 1. Go to https://aws.amazon.com/
# 2. Click "Create an AWS Account"
# 3. Follow the registration process
# 4. Enable MFA on root account (Security → MFA)
```

**Create IAM User for Terraform**:
```bash
# In AWS Console:
# 1. Go to IAM → Users → Add User
# 2. User name: terraform-admin
# 3. Enable "Programmatic access"
# 4. Attach policies:
#    - AdministratorAccess (for initial setup)
#    - Or create custom policy (more secure)
# 5. Save Access Key ID and Secret Access Key
```

**Configure AWS CLI**:
```bash
# Install AWS CLI
brew install awscli  # macOS
# OR
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials
aws configure
# AWS Access Key ID: [paste your key]
# AWS Secret Access Key: [paste your secret]
# Default region: us-east-1
# Default output format: json

# Verify
aws sts get-caller-identity
```

### 2. Install Required Tools

**Terraform**:
```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Linux
wget https://releases.hashicorp.com/terraform/1.5.0/terraform_1.5.0_linux_amd64.zip
unzip terraform_1.5.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Verify
terraform version
```

**kubectl**:
```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

**Helm**:
```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

**Docker**:
```bash
# macOS
brew install --cask docker

# Linux (Ubuntu)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Verify
docker version
```

**Additional Tools**:
```bash
# k9s (Kubernetes CLI)
brew install k9s

# jq (JSON processor)
brew install jq

# yq (YAML processor)
brew install yq

# aws-iam-authenticator
brew install aws-iam-authenticator
```

### 3. GitHub Setup

**Create Repository**:
```bash
# 1. Go to GitHub.com
# 2. Click "New Repository"
# 3. Name: devops-platform
# 4. Initialize with README
# 5. Clone locally

git clone git@github.com:YOUR_USERNAME/devops-platform.git
cd devops-platform
```

**Set Up GitHub Secrets** (for CI/CD later):
```
Repository Settings → Secrets and variables → Actions

Add secrets:
- AWS_REGION: us-east-1
- AWS_ACCOUNT_ID: [your account ID]
```

### 4. Domain Name (Optional but Recommended)

```bash
# Purchase domain from Route53 or external registrar
# We'll use: example.com
# Update DNS later in Phase 2
```

---

## Phase 1: Foundation Setup

### Step 1.1: Initialize Project Structure

```bash
# Create directory structure
mkdir -p {terraform/{modules,environments/{dev,prod},global},k8s/{base,overlays/{dev,prod}},apps,scripts,.github/workflows}

cd terraform

# Directory structure should look like:
# terraform/
# ├── modules/
# ├── environments/
# │   ├── dev/
# │   └── prod/
# └── global/
```

### Step 1.2: Set Up Terraform Backend

```bash
# Create script to initialize S3 backend
cat > scripts/init-backend.sh << 'EOF'
#!/bin/bash
set -e

AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION="us-east-1"
BUCKET_NAME="terraform-state-${AWS_ACCOUNT_ID}"
DYNAMODB_TABLE="terraform-state-lock"

echo "Creating Terraform backend..."
echo "Bucket: ${BUCKET_NAME}"
echo "Region: ${AWS_REGION}"

# Create S3 bucket
aws s3 mb "s3://${BUCKET_NAME}" --region "${AWS_REGION}" 2>/dev/null || echo "Bucket already exists"

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

# Create DynamoDB table
aws dynamodb create-table \
  --table-name "${DYNAMODB_TABLE}" \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region "${AWS_REGION}" 2>/dev/null || echo "DynamoDB table already exists"

echo "✅ Terraform backend initialized successfully!"
echo "Backend config:"
echo "  bucket: ${BUCKET_NAME}"
echo "  region: ${AWS_REGION}"
echo "  dynamodb_table: ${DYNAMODB_TABLE}"
EOF

chmod +x scripts/init-backend.sh

# Run the script
./scripts/init-backend.sh
```

### Step 1.3: Create Global Resources

**GitHub OIDC Provider** (for secure CI/CD):

```bash
# Create terraform/global/iam/github-oidc.tf
mkdir -p terraform/global/iam
cat > terraform/global/iam/main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    key            = "global/iam/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_caller_identity" "current" {}

# OIDC Provider for GitHub Actions
data "tls_certificate" "github" {
  url = "https://token.actions.githubusercontent.com/.well-known/openid-configuration"
}

resource "aws_iam_openid_connect_provider" "github_actions" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.github.certificates[0].sha1_fingerprint]

  tags = {
    Name = "github-actions-oidc"
  }
}

# IAM role for GitHub Actions
resource "aws_iam_role" "github_actions" {
  name = "github-actions-deployment-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github_actions.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            "token.actions.githubusercontent.com:sub" = "repo:YOUR_GITHUB_ORG/devops-platform:*"
          }
        }
      }
    ]
  })
}

# Attach necessary policies
resource "aws_iam_role_policy_attachment" "github_ecr" {
  role       = aws_iam_role.github_actions.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser"
}

resource "aws_iam_role_policy" "github_eks" {
  name = "eks-access"
  role = aws_iam_role.github_actions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "eks:DescribeCluster",
          "eks:ListClusters"
        ]
        Resource = "*"
      }
    ]
  })
}

output "github_actions_role_arn" {
  description = "ARN of IAM role for GitHub Actions"
  value       = aws_iam_role.github_actions.arn
}
EOF

# Update with your GitHub org/repo
sed -i '' 's/YOUR_GITHUB_ORG/your-username/g' terraform/global/iam/main.tf

# Initialize and apply
cd terraform/global/iam
terraform init -backend-config="bucket=terraform-state-$(aws sts get-caller-identity --query Account --output text)"
terraform plan
terraform apply

# Save the output
terraform output github_actions_role_arn
cd ../../..
```

**ECR Repositories**:

```bash
mkdir -p terraform/global/ecr
cat > terraform/global/ecr/main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
  
  backend "s3" {
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
    "user-service"
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
        description  = "Keep last 10 images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v", "main-"]
          countType     = "imageCountMoreThan"
          countNumber   = 10
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2
        description  = "Remove untagged after 7 days"
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
  value = { for k, v in aws_ecr_repository.repos : k => v.repository_url }
}
EOF

cd terraform/global/ecr
terraform init -backend-config="bucket=terraform-state-$(aws sts get-caller-identity --query Account --output text)"
terraform apply
cd ../../..
```

---

## Phase 2: Infrastructure Provisioning

### Step 2.1: Download Terraform Modules

Create reusable Terraform modules. For brevity, we'll use the official AWS modules with our configurations:

```bash
# Create dev environment configuration
mkdir -p terraform/environments/dev

cat > terraform/environments/dev/main.tf << 'EOF'
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
  }
  
  backend "s3" {
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = "dev"
      ManagedBy   = "Terraform"
      Project     = "DevOps Platform"
    }
  }
}

locals {
  environment = "dev"
  azs         = ["us-east-1a", "us-east-1b"]
}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${local.environment}-vpc"
  cidr = "10.1.0.0/16"

  azs             = local.azs
  public_subnets  = ["10.1.1.0/24", "10.1.2.0/24"]
  private_subnets = ["10.1.11.0/24", "10.1.12.0/24"]
  intra_subnets   = ["10.1.21.0/24", "10.1.22.0/24"]
  database_subnets = ["10.1.31.0/24", "10.1.32.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = true  # Cost optimization
  one_nat_gateway_per_az = false

  enable_dns_hostnames = true
  enable_dns_support   = true

  # Tags for EKS
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
    "kubernetes.io/cluster/${local.environment}-eks-cluster" = "shared"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
    "kubernetes.io/cluster/${local.environment}-eks-cluster" = "shared"
  }
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "${local.environment}-eks-cluster"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  control_plane_subnet_ids = module.vpc.intra_subnets

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  # Node groups
  eks_managed_node_groups = {
    general = {
      name           = "${local.environment}-general-ng"
      instance_types = ["t3.medium"]
      capacity_type  = "SPOT"
      
      min_size     = 1
      max_size     = 5
      desired_size = 2

      labels = {
        Environment = local.environment
      }
      
      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            volume_size           = 50
            volume_type           = "gp3"
            delete_on_termination = true
          }
        }
      }
    }
  }

  # Cluster add-ons
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
  }
}

# RDS PostgreSQL
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "${local.environment}-postgres"

  engine               = "postgres"
  engine_version       = "15.3"
  family               = "postgres15"
  major_engine_version = "15"
  instance_class       = "db.t3.micro"

  allocated_storage     = 20
  max_allocated_storage = 100
  storage_encrypted     = true

  db_name  = "devdb"
  username = "dbadmin"
  port     = 5432

  multi_az               = false
  db_subnet_group_name   = module.vpc.database_subnet_group_name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = 1
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  skip_final_snapshot = true
}

# Security Groups
resource "aws_security_group" "rds" {
  name_prefix = "${local.environment}-rds-"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = module.vpc.private_subnets_cidr_blocks
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Bastion Host
resource "aws_instance" "bastion" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.micro"
  
  subnet_id                   = module.vpc.public_subnets[0]
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
              EOF

  tags = {
    Name = "${local.environment}-bastion"
  }
}

resource "aws_key_pair" "bastion" {
  key_name   = "${local.environment}-bastion-key"
  public_key = file("~/.ssh/id_rsa.pub")  # Use your SSH public key
}

resource "aws_security_group" "bastion" {
  name_prefix = "${local.environment}-bastion-"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.bastion_allowed_cidrs
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
EOF

# Variables
cat > terraform/environments/dev/variables.tf << 'EOF'
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "bastion_allowed_cidrs" {
  description = "CIDR blocks allowed to SSH to bastion"
  type        = list(string)
  default     = ["0.0.0.0/0"]  # Update with your IP!
}
EOF

# Outputs
cat > terraform/environments/dev/outputs.tf << 'EOF'
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "eks_cluster_endpoint" {
  value     = module.eks.cluster_endpoint
  sensitive = true
}

output "eks_cluster_name" {
  value = module.eks.cluster_name
}

output "rds_endpoint" {
  value     = module.rds.db_instance_endpoint
  sensitive = true
}

output "bastion_public_ip" {
  value = aws_instance.bastion.public_ip
}

output "configure_kubectl" {
  value = "aws eks update-kubeconfig --region ${var.aws_region} --name ${module.eks.cluster_name}"
}
EOF
```

### Step 2.2: Deploy Development Infrastructure

```bash
cd terraform/environments/dev

# Initialize Terraform
terraform init -backend-config="bucket=terraform-state-$(aws sts get-caller-identity --query Account --output text)"

# Review the plan
terraform plan

# Apply (this will take 15-20 minutes)
terraform apply

# Save outputs
terraform output -json > outputs.json

# Configure kubectl
eval $(terraform output -raw configure_kubectl)

# Verify cluster access
kubectl get nodes
kubectl get namespaces
```

Expected output:
```
NAME                                       STATUS   ROLES    AGE   VERSION
ip-10-1-11-xxx.ec2.internal               Ready    <none>   2m    v1.28.x
ip-10-1-12-xxx.ec2.internal               Ready    <none>   2m    v1.28.x
```

---

## Phase 3: Application Deployment

### Step 3.1: Install Kubernetes Add-ons

**AWS Load Balancer Controller**:

```bash
# Create IAM policy
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Get cluster info
CLUSTER_NAME=$(terraform output -raw eks_cluster_name)
AWS_REGION="us-east-1"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create IAM role for service account
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

# Install controller using Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify installation
kubectl get deployment -n kube-system aws-load-balancer-controller
```

**EBS CSI Driver**:

```bash
# Create IAM role
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

# Install addon
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole

# Verify
kubectl get pods -n kube-system -l app=ebs-csi-controller
```

### Step 3.2: Install Kafka (Strimzi)

```bash
# Create namespace
kubectl create namespace kafka

# Install Strimzi operator
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

# Wait for operator to be ready
kubectl wait --for=condition=ready pod -l name=strimzi-cluster-operator -n kafka --timeout=300s

# Create Kafka cluster
cat > kafka-cluster.yaml << 'EOF'
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: dev-kafka
  namespace: kafka
spec:
  kafka:
    version: 3.5.1
    replicas: 2
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 2
      transaction.state.log.replication.factor: 2
      transaction.state.log.min.isr: 1
      default.replication.factor: 2
      min.insync.replicas: 1
    storage:
      type: persistent-claim
      size: 50Gi
      class: gp3
    resources:
      requests:
        memory: 2Gi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: 1000m
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 10Gi
      class: gp3
    resources:
      requests:
        memory: 1Gi
        cpu: 500m
      limits:
        memory: 2Gi
        cpu: 1000m
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF

kubectl apply -f kafka-cluster.yaml

# Wait for Kafka to be ready (5-10 minutes)
kubectl wait kafka/dev-kafka --for=condition=Ready --timeout=600s -n kafka

# Verify
kubectl get kafka -n kafka
kubectl get pods -n kafka
```

### Step 3.3: Create Sample Application Namespaces

```bash
# Create namespace for dev applications
kubectl create namespace dev

# Create storage class for apps
cat > storageclass.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF

kubectl apply -f storageclass.yaml

# Set as default
kubectl patch storageclass gp3 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Step 3.4: Deploy Sample NestJS Application

```bash
# Create simple NestJS app deployment
cat > sample-api-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-api
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-api
  template:
    metadata:
      labels:
        app: sample-api
    spec:
      containers:
      - name: api
        image: nginx:latest  # Replace with your app image
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: sample-api
  namespace: dev
spec:
  selector:
    app: sample-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-api-ingress
  namespace: dev
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sample-api
            port:
              number: 80
EOF

kubectl apply -f sample-api-deployment.yaml

# Wait for deployment
kubectl rollout status deployment/sample-api -n dev

# Get ALB URL
kubectl get ingress -n dev sample-api-ingress
```

---

## Phase 4: CI/CD Configuration

### Step 4.1: Set Up GitHub Actions Workflows

```bash
# Create workflow directory
mkdir -p .github/workflows

# Create CI workflow
cat > .github/workflows/ci.yml << 'EOF'
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Placeholder test
        run: echo "Tests would run here"
EOF

# Create build and deploy workflow
cat > .github/workflows/deploy-dev.yml << 'EOF'
name: Deploy to Dev

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --region ${{ env.AWS_REGION }} \
            --name dev-eks-cluster

      - name: Deploy to dev
        run: |
          kubectl apply -f sample-api-deployment.yaml
          kubectl rollout status deployment/sample-api -n dev
EOF

# Commit and push
git add .github/
git commit -m "Add CI/CD workflows"
git push
```

### Step 4.2: Update GitHub Secrets

```bash
# Get the GitHub Actions role ARN from Terraform
cd terraform/global/iam
ROLE_ARN=$(terraform output -raw github_actions_role_arn)
echo $ROLE_ARN

# Add to GitHub:
# 1. Go to your repository on GitHub
# 2. Settings → Secrets and variables → Actions
# 3. Add secret: AWS_ROLE_ARN = [paste ARN]
```

---

## Phase 5: Observability Setup

### Step 5.1: Install Prometheus

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create values file
cat > prometheus-values.yaml << 'EOF'
server:
  retention: 15d
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 1000m
      memory: 4Gi
  persistentVolume:
    size: 50Gi
    storageClass: gp3

alertmanager:
  enabled: true
  persistentVolume:
    size: 10Gi
    storageClass: gp3

pushgateway:
  enabled: false

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
EOF

# Install Prometheus
kubectl create namespace monitoring
helm install prometheus prometheus-community/prometheus \
  -n monitoring \
  -f prometheus-values.yaml

# Verify
kubectl get pods -n monitoring
```

### Step 5.2: Install Loki

```bash
# Add Grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create values file
cat > loki-values.yaml << 'EOF'
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  storage:
    type: filesystem

singleBinary:
  replicas: 1
  persistence:
    enabled: true
    size: 30Gi
    storageClass: gp3

monitoring:
  selfMonitoring:
    enabled: false
  lokiCanary:
    enabled: false
EOF

# Install Loki
helm install loki grafana/loki \
  -n monitoring \
  -f loki-values.yaml

# Install Promtail
cat > promtail-values.yaml << 'EOF'
config:
  lokiAddress: http://loki:3100/loki/api/v1/push
EOF

helm install promtail grafana/promtail \
  -n monitoring \
  -f promtail-values.yaml

# Verify
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki
```

### Step 5.3: Install Grafana

```bash
cat > grafana-values.yaml << 'EOF'
adminPassword: admin  # Change this!

persistence:
  enabled: true
  size: 10Gi
  storageClassName: gp3

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server
      isDefault: true
    - name: Loki
      type: loki
      url: http://loki:3100

dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards/default

dashboards:
  default:
    kubernetes-cluster:
      gnetId: 7249
      datasource: Prometheus
    node-exporter:
      gnetId: 1860
      datasource: Prometheus
EOF

helm install grafana grafana/grafana \
  -n monitoring \
  -f grafana-values.yaml

# Get Grafana password
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# Port forward to access Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80

# Access at http://localhost:3000
# Username: admin
# Password: [from command above]
```

---

## Phase 6: Production Deployment

### Step 6.1: Create Production Environment

```bash
# Copy dev configuration to prod
cp -r terraform/environments/dev terraform/environments/prod

# Update prod configuration
cd terraform/environments/prod

# Modify main.tf for production settings:
# - Change CIDR to 10.0.0.0/16
# - Enable multi-AZ (3 AZs)
# - Use on-demand + spot mix
# - Enable Aurora instead of RDS
# - Increase instance sizes

# Update backend key in main.tf
sed -i '' 's/key            = "dev/key            = "prod/g' main.tf

# Update locals
sed -i '' 's/environment = "dev"/environment = "prod"/g' main.tf
sed -i '' 's/10.1./10.0./g' main.tf

# Initialize and apply
terraform init -backend-config="bucket=terraform-state-$(aws sts get-caller-identity --query Account --output text)"
terraform plan
terraform apply  # Takes 20-30 minutes

# Configure kubectl for prod
aws eks update-kubeconfig --region us-east-1 --name prod-eks-cluster

# Verify
kubectl get nodes
```

### Step 6.2: Deploy to Production

```bash
# Create production namespace
kubectl create namespace production

# Install same add-ons as dev (LoadBalancer Controller, EBS CSI)
# [Repeat steps from Phase 3.1]

# Deploy applications with production configuration
# [Adjust resource limits, replicas, etc.]
```

---

## Phase 7: Operations and Maintenance

### Step 7.1: Set Up Monitoring Alerts

```bash
# Configure Grafana alerts
# 1. Go to Grafana → Alerting → Contact points
# 2. Add email/Slack notification channel
# 3. Create alerts for:
#    - High CPU usage (> 80%)
#    - High memory usage (> 85%)
#    - Pod crash loops
#    - PVC usage (> 80%)
```

### Step 7.2: Create Backup Procedures

```bash
# Automated EBS snapshots
cat > backup-cron.yaml << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-volumes
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-sa
          containers:
          - name: backup
            image: amazon/aws-cli
            command:
            - /bin/sh
            - -c
            - |
              aws ec2 describe-volumes \
                --filters "Name=tag:Environment,Values=dev" \
                --query 'Volumes[*].VolumeId' \
                --output text | xargs -n1 aws ec2 create-snapshot --volume-id
          restartPolicy: OnFailure
EOF

kubectl apply -f backup-cron.yaml
```

### Step 7.3: Documentation

```bash
# Create operational runbooks
mkdir -p docs/runbooks

cat > docs/runbooks/scale-deployment.md << 'EOF'
# Scaling Deployments

## Scale up
```bash
kubectl scale deployment/sample-api -n dev --replicas=5
```

## Scale down
```bash
kubectl scale deployment/sample-api -n dev --replicas=2
```

## Auto-scaling
```bash
kubectl autoscale deployment/sample-api -n dev --min=2 --max=10 --cpu-percent=70
```
EOF
```

---

## Troubleshooting

### Common Issues

**Issue 1: EKS nodes not joining cluster**

```bash
# Check nodes
kubectl get nodes

# Check node logs
aws ec2 describe-instances --filters "Name=tag:eks:cluster-name,Values=dev-eks-cluster"

# Check IAM roles
aws eks describe-cluster --name dev-eks-cluster --query cluster.roleArn
```

**Issue 2: Pods stuck in Pending**

```bash
# Check events
kubectl describe pod POD_NAME -n dev

# Check resources
kubectl top nodes
kubectl describe node NODE_NAME

# Common causes:
# - Insufficient resources
# - PVC not bound
# - Node selector mismatch
```

**Issue 3: Cannot access services**

```bash
# Check service
kubectl get svc -n dev
kubectl describe svc SERVICE_NAME -n dev

# Check ingress
kubectl get ingress -n dev
kubectl describe ingress INGRESS_NAME -n dev

# Check Load Balancer Controller
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

**Issue 4: High costs**

```bash
# Check AWS Cost Explorer
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# Common issues:
# - NAT Gateway data processing
# - Running resources when not needed
# - Wrong instance types
# - Unused EBS volumes/snapshots
```

---

## Validation Checklist

### Infrastructure

- [ ] VPC created with proper subnets
- [ ] EKS cluster accessible via kubectl
- [ ] Nodes joined and ready
- [ ] NAT Gateway functional (nodes can access internet)
- [ ] Security groups properly configured
- [ ] RDS accessible from EKS
- [ ] Bastion host accessible via SSH

### Applications

- [ ] Kafka cluster running
- [ ] Sample API deployed
- [ ] Ingress working (ALB created)
- [ ] Services responding to requests
- [ ] Persistent volumes bound
- [ ] Resource limits enforced

### CI/CD

- [ ] GitHub Actions workflows running
- [ ] OIDC authentication working
- [ ] ECR images pushed successfully
- [ ] Deployments triggered on push
- [ ] Rollback mechanism tested

### Monitoring

- [ ] Prometheus collecting metrics
- [ ] Loki collecting logs
- [ ] Grafana accessible
- [ ] Dashboards showing data
- [ ] Alerts configured

### Security

- [ ] IAM roles following least privilege
- [ ] Secrets in AWS Secrets Manager
- [ ] Network policies applied
- [ ] Security groups restrictive
- [ ] Encryption enabled (EBS, RDS)

### Cost

- [ ] Budget alerts configured
- [ ] Spot instances running
- [ ] Auto-scaling configured
- [ ] Unused resources cleaned up
- [ ] Cost tags applied

---

## Next Steps

After completing this guide:

1. **Customize** the infrastructure for your specific needs
2. **Deploy** your actual applications
3. **Configure** production CI/CD pipelines
4. **Set up** alerting and on-call procedures
5. **Document** your specific operational procedures
6. **Train** team members on the infrastructure
7. **Perform** disaster recovery drills
8. **Optimize** costs continuously
9. **Stay updated** with AWS and Kubernetes releases
10. **Scale** as your traffic grows

### Learning Resources

- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Terraform AWS Provider Docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [NestJS Documentation](https://docs.nestjs.com/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)

---

## Summary

You now have:

✅ Complete AWS infrastructure (VPC, EKS, RDS, etc.)
✅ Kubernetes cluster with essential add-ons
✅ Kafka event streaming platform
✅ Sample application deployed
✅ CI/CD pipelines configured
✅ Full observability stack (Prometheus, Loki, Grafana)
✅ Both dev and production environments
✅ Security best practices implemented
✅ Cost optimization strategies in place

**Congratulations!** You've built a production-grade DevOps platform on AWS! 🎉

---

[← Back to Cost Analysis](./19-cost-analysis.md) | [Next: Troubleshooting Guide →](./22-troubleshooting-guide.md)
