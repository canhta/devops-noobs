---
agent: agent
---

I want to design and implement a complete end-to-end DevOps infrastructure on AWS, similar in scale and structure to the sample system described below. Your task is to research, design, and document a full system architecture and produce a detailed, step-by-step guide covering every aspect.
The system will host a microservices application (NestJS services running in multiple pods) using Kafka, with CI/CD, observability, autoscaling, monitoring, security, and full environment separation. All infrastructure must be provisioned using Terraform, and all deployments must run on AWS.

Objectives
Produce a complete technical architecture and guide that includes:

- AWS infrastructure design
- Kubernetes cluster (EKS)
- Kafka (managed MSK or self-managed — include recommendation)
- CI/CD with GitHub Actions
- Logging and monitoring (Prometheus, Loki, Grafana)
- Autoscaling (HPA + Cluster Autoscaler or Karpenter)
- Release flows: manual and automatic
- Dev/Prod environment separation
- Terraform IaC with preview capabilities on GitHub
- Bastion host setup
- Cost estimation
- Operations, security, DR, scaling
- Step-by-step instructions for every component
  This should become a complete blueprint similar in detail to the “Training Central” system example below, but adapted to my requirements.

System Requirements

1. Architecture

- Cloud-native microservices architecture on AWS
- Kubernetes orchestrated via EKS
- Kafka as the event backbone for the NestJS services
- IaC using Terraform with modular structure
- CI/CD using GitHub Actions
- Optional: GitOps flow comparison (ArgoCD) with explanation
- Two environments: dev and prod
  Deliverables must include:
- Component architecture diagram
- Network architecture diagram
- Data flow diagram
- Deployment workflow diagram
- CI/CD pipeline diagram

2. Infrastructure Components
   Amazon EKS

- Cluster per environment (or shared cluster with namespace isolation — analyze tradeoffs)
- Node groups (spot/on-demand mix)
- EBS CSI driver
- Control plane in dedicated subnets
  Kafka
- MSK (recommended for production)
- OR Strimzi operator in Kubernetes (recommended for learning)
- Include tradeoffs, cost, and operational complexity comparison
  Networking
- VPC with 3 AZs
- Public, private, and control-plane subnets
- NAT gateway, IGW, route tables
- VPC endpoints for secure internal access
  Compute
- Bastion host with minimal access and preinstalled tools
- EKS node groups with autoscaling
  Container Registry
- ECR repositories for all services
- Immutable tags
- Image scanning
- Lifecycle rules
  Storage
- S3 for assets or backups
- EBS for pods
- Optional: RDS/Aurora depending on service requirements

3. Security Model
   Network Security

- SGs for ALB, nodes, bastion, Kafka
- Private-only workloads
- NACLs best practices
  IAM
- Pod-level IAM roles (IRSA)
- GitHub OIDC integration
- Minimal-permission roles
  Data Security
- Full encryption at rest + in transit
- Logging practices
- Container image scanning
  Access Control
- Bastion host with restricted SSH
- RBAC on Kubernetes
- Secret management (AWS Secrets Manager or External Secrets Operator)

4. Application & Service Architecture

- NestJS microservices (deployment patterns)
- Kafka consumers/producers
- API gateway or ingress controller
- Req/resp and event-driven service interaction diagrams
- Namespace & resource isolation strategy

5. Deployment & CI/CD
   CI/CD via GitHub Actions
   Must include:
   Build pipeline

- Code checkout
- Testing
- Docker build & push to ECR
  Deployment pipeline
- Auto-deploy to dev on main branch
- Manual “promote to prod” workflow with approval
- Kubernetes manifests validation
- Terraform plan & apply preview in PRs (comment to PR)
  Release Flows
- Automatic dev updates on merge
- Manual prod updates
- Rollback mechanisms
- Versioning strategy

6. Observability & Logging
   Provide full integration setup:
   Prometheus

- Node exporter
- kube-state-metrics
- Application metric endpoints
  Loki
- Promtail or Fluent Bit daemonset
- Log retention strategy
  Grafana
- Dashboards for:
  - Kafka lag
  - NestJS API performance
  - Pod resource usage
  - Cluster health

7. Autoscaling

- Horizontal Pod Autoscaler (CPU, memory, custom metrics)
- Cluster autoscaling (Cluster Autoscaler vs Karpenter — include recommendation)
- Load testing scenarios and expected scaling behavior

8. Cost Analysis
   Provide a detailed monthly estimate including:

- EKS control plane
- Worker nodes (spot vs on-demand)
- MSK vs self-managed Kafka
- NAT Gateway
- ECR
- S3
- Networking
- Monitoring stack
  Also include cost optimization strategies.

9. Operations & Maintenance

- Monitoring procedures
- Backup strategies
- Maintenance schedules
- Security patches
- Terraform state management
- Log rotation
- Support scripts

10. Disaster Recovery

- RTO & RPO for all components
- Recovery procedures
- Infrastructure rebuild using Terraform
- Data restore from S3/RDS/EBS
- Multi-AZ and multi-region considerations

11. Scaling Strategy

- Horizontal & vertical scaling at both node and pod levels
- Data scaling for Kafka
- Geographic expansion strategy
- CDN or Route 53 for global routing

Expected Output
You must produce:

1. A full system blueprint, following the structure above.
2. Complete step-by-step guide, extremely detailed, suitable for a DevOps beginner to learn.
3. Architecture diagrams (ASCII or Mermaid).
4. Terraform structure plan (modules, folders, variables).
5. CI/CD YAML examples for GitHub Actions.
6. Kubernetes examples (Manifests or Helm/Kustomize structure).
7. Cost table + optimization notes.
8. Operational procedures and checklists.
   The output should be as comprehensive as a technical design document used for onboarding DevOps engineers.
