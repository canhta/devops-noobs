# 11. Container Registry (Amazon ECR)

## Table of Contents
1. [Overview](#overview)
2. [ECR Setup](#ecr-setup)
3. [Repository Management](#repository-management)
4. [Image Lifecycle Policies](#image-lifecycle-policies)
5. [Image Scanning](#image-scanning)
6. [Multi-Region Replication](#multi-region-replication)
7. [Access Control](#access-control)
8. [CI/CD Integration](#cicd-integration)
9. [Image Optimization](#image-optimization)
10. [Best Practices](#best-practices)

---

## Overview

Amazon Elastic Container Registry (ECR) serves as the private Docker registry for storing, managing, and deploying container images.

### ECR Benefits

- **Integrated**: Native integration with EKS, ECS, and AWS services
- **Secure**: Encryption at rest/transit, image scanning
- **Scalable**: Highly available, managed by AWS
- **Cost-Effective**: Pay only for storage and data transfer

### Repository Structure

```
AWS Account (ACCOUNT_ID)
├── api-gateway
│   ├── latest
│   ├── v1.2.3
│   └── dev-abc123
├── auth-service
│   ├── latest
│   ├── v1.0.5
│   └── prod-xyz789
├── order-service
├── user-service
└── notification-service
```

---

## ECR Setup

### Terraform Configuration

```hcl
# modules/ecr/main.tf

locals {
  repositories = [
    "api-gateway",
    "auth-service",
    "order-service",
    "user-service",
    "notification-service"
  ]
}

resource "aws_ecr_repository" "services" {
  for_each = toset(local.repositories)
  
  name                 = each.value
  image_tag_mutability = "MUTABLE"  # or "IMMUTABLE" for prod

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name        = each.value
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# KMS Key for encryption
resource "aws_kms_key" "ecr" {
  description             = "ECR encryption key"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  tags = {
    Name = "${var.environment}-ecr-key"
  }
}

resource "aws_kms_alias" "ecr" {
  name          = "alias/${var.environment}-ecr"
  target_key_id = aws_kms_key.ecr.key_id
}

# Output repository URLs
output "repository_urls" {
  value = {
    for repo in aws_ecr_repository.services :
    repo.name => repo.repository_url
  }
}
```

### Create Repositories via CLI

```bash
# Create single repository
aws ecr create-repository \
  --repository-name api-gateway \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:ACCOUNT_ID:key/KEY_ID

# Enable image scanning
aws ecr put-image-scanning-configuration \
  --repository-name api-gateway \
  --image-scanning-configuration scanOnPush=true
```

---

## Repository Management

### Login to ECR

```bash
# Get login token and authenticate Docker
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# For older AWS CLI versions
$(aws ecr get-login --no-include-email --region us-east-1)
```

### Push Images

```bash
# Build image
docker build -t api-gateway:v1.2.3 .

# Tag for ECR
docker tag api-gateway:v1.2.3 \
  ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/api-gateway:v1.2.3

docker tag api-gateway:v1.2.3 \
  ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/api-gateway:latest

# Push to ECR
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/api-gateway:v1.2.3
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/api-gateway:latest
```

### List Images

```bash
# List all repositories
aws ecr describe-repositories

# List images in repository
aws ecr describe-images --repository-name api-gateway

# List images with specific tag
aws ecr describe-images \
  --repository-name api-gateway \
  --image-ids imageTag=v1.2.3
```

### Delete Images

```bash
# Delete specific image tag
aws ecr batch-delete-image \
  --repository-name api-gateway \
  --image-ids imageTag=v1.0.0

# Delete by digest
aws ecr batch-delete-image \
  --repository-name api-gateway \
  --image-ids imageDigest=sha256:abc123...
```

---

## Image Lifecycle Policies

### Lifecycle Policy JSON

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 production images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Expire dev images older than 7 days",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["dev-", "feature-"],
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 3,
      "description": "Expire untagged images after 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

### Apply Lifecycle Policy

```hcl
# Terraform
resource "aws_ecr_lifecycle_policy" "api_gateway" {
  repository = aws_ecr_repository.services["api-gateway"].name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 production images"
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
        description  = "Expire dev images older than 7 days"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["dev-", "feature-"]
          countType     = "sinceImagePushed"
          countUnit     = "days"
          countNumber   = 7
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 3
        description  = "Expire untagged images after 1 day"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 1
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

```bash
# AWS CLI
aws ecr put-lifecycle-policy \
  --repository-name api-gateway \
  --lifecycle-policy-text file://lifecycle-policy.json
```

---

## Image Scanning

### Enable Scanning

```hcl
# Terraform - already included in repository configuration
image_scanning_configuration {
  scan_on_push = true
}
```

### Trigger Manual Scan

```bash
# Scan specific image
aws ecr start-image-scan \
  --repository-name api-gateway \
  --image-id imageTag=v1.2.3

# Check scan status
aws ecr describe-image-scan-findings \
  --repository-name api-gateway \
  --image-id imageTag=v1.2.3
```

### View Scan Results

```bash
# Get scan findings
aws ecr describe-image-scan-findings \
  --repository-name api-gateway \
  --image-id imageTag=v1.2.3 \
  --query 'imageScanFindings.findings[?severity==`CRITICAL`]'

# Count vulnerabilities by severity
aws ecr describe-image-scan-findings \
  --repository-name api-gateway \
  --image-id imageTag=v1.2.3 \
  --query 'imageScanFindings.findingSeverityCounts'
```

### Scan in CI/CD

```yaml
# .github/workflows/build.yml
- name: Scan Docker image
  run: |
    # Wait for scan to complete
    aws ecr wait image-scan-complete \
      --repository-name api-gateway \
      --image-id imageTag=${{ github.sha }}
    
    # Get scan results
    SCAN_FINDINGS=$(aws ecr describe-image-scan-findings \
      --repository-name api-gateway \
      --image-id imageTag=${{ github.sha }} \
      --query 'imageScanFindings.findingSeverityCounts')
    
    echo "Scan findings: $SCAN_FINDINGS"
    
    # Fail if critical vulnerabilities found
    CRITICAL=$(echo $SCAN_FINDINGS | jq -r '.CRITICAL // 0')
    if [ $CRITICAL -gt 0 ]; then
      echo "Found $CRITICAL critical vulnerabilities"
      exit 1
    fi
```

---

## Multi-Region Replication

### Replication Configuration

```hcl
# modules/ecr/replication.tf

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

    rule {
      destination {
        region      = "eu-west-1"
        registry_id = data.aws_caller_identity.current.account_id
      }

      repository_filter {
        filter      = "prod-"
        filter_type = "PREFIX_MATCH"
      }
    }
  }
}
```

### Cross-Region Pull

```bash
# Pull from secondary region
aws ecr get-login-password --region us-west-2 | \
  docker login --username AWS --password-stdin \
  ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com

docker pull ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/api-gateway:v1.2.3
```

---

## Access Control

### Repository Policy

```hcl
resource "aws_ecr_repository_policy" "api_gateway" {
  repository = aws_ecr_repository.services["api-gateway"].name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowPull"
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/eks-${var.environment}-node-role",
            "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/github-actions-role"
          ]
        }
        Action = [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability"
        ]
      },
      {
        Sid    = "AllowPush"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/github-actions-role"
        }
        Action = [
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload"
        ]
      }
    ]
  })
}
```

### IAM Policy for EKS Nodes

```hcl
resource "aws_iam_role_policy_attachment" "eks_ecr_read" {
  role       = aws_iam_role.eks_nodes.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}
```

---

## CI/CD Integration

### GitHub Actions Workflow

```yaml
name: Build and Push to ECR

on:
  push:
    branches: [main, develop]

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com
  ECR_REPOSITORY: api-gateway

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2
    
    - name: Build, tag, and push image
      env:
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
    
    - name: Scan image for vulnerabilities
      run: |
        aws ecr start-image-scan \
          --repository-name $ECR_REPOSITORY \
          --image-id imageTag=${{ github.sha }}
        
        aws ecr wait image-scan-complete \
          --repository-name $ECR_REPOSITORY \
          --image-id imageTag=${{ github.sha }}
```

---

## Image Optimization

### Multi-Stage Build

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy only necessary files
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

USER nodejs
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Image Size Optimization

```dockerfile
# Use alpine variants
FROM node:20-alpine  # vs node:20 (900MB vs 180MB)

# Remove unnecessary files
RUN npm ci --only=production && \
    npm cache clean --force

# Use .dockerignore
# .dockerignore file:
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.DS_Store
```

### Caching Strategies

```dockerfile
# Leverage layer caching
COPY package*.json ./
RUN npm ci  # This layer will be cached if package.json doesn't change

COPY . .    # Only rebuild this if source code changes
RUN npm run build
```

---

## Best Practices

### 1. Image Tagging Strategy

```bash
# Use semantic versioning for releases
v1.2.3

# Include git commit for traceability
v1.2.3-abc1234

# Environment-specific tags
dev-latest
staging-v1.2.3
prod-v1.2.3

# Never use 'latest' in production
# Always pin to specific versions in K8s manifests
```

### 2. Security Scanning

- ✅ Enable scan on push
- ✅ Fail builds with critical vulnerabilities
- ✅ Regular scan of existing images
- ✅ Use minimal base images (alpine, distroless)

### 3. Lifecycle Management

- ✅ Keep last 10 production images
- ✅ Delete dev/feature branch images after 7 days
- ✅ Remove untagged images after 1 day
- ✅ Regular cleanup to reduce costs

### 4. Access Control

- ✅ Use IAM roles, not access keys
- ✅ Separate read/write permissions
- ✅ Repository-level policies
- ✅ Enable encryption at rest

### 5. Cost Optimization

```bash
# Monitor ECR storage costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --filter file://ecr-filter.json

# ecr-filter.json
{
  "Dimensions": {
    "Key": "SERVICE",
    "Values": ["Amazon Elastic Container Registry"]
  }
}
```

---

[← Back to CI/CD Pipeline](./10-cicd-pipeline.md) | [Next: Observability Stack →](./12-observability-stack.md)
