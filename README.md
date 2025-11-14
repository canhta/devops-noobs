# Enterprise DevOps Infrastructure on AWS

A complete end-to-end DevOps infrastructure blueprint for deploying microservices applications on AWS using Kubernetes (EKS), Kafka, and modern observability tools.

## 🎯 Project Overview

This project provides a production-ready, cloud-native infrastructure design for hosting microservices applications (NestJS) with event-driven architecture using Kafka. It includes complete automation, monitoring, security, and operational procedures suitable for enterprise-grade deployments.

### Key Features

- **Cloud Provider**: Amazon Web Services (AWS)
- **Container Orchestration**: Amazon EKS (Elastic Kubernetes Service)
- **Event Streaming**: Apache Kafka (MSK or Strimzi)
- **Infrastructure as Code**: Terraform with modular design
- **CI/CD**: GitHub Actions with environment separation
- **Observability**: Prometheus, Loki, Grafana stack
- **Auto-scaling**: HPA + Cluster Autoscaler/Karpenter
- **Environments**: Dev and Production with proper isolation

## 📚 Documentation Structure

### Core Architecture
- **[01. System Architecture Overview](./docs/01-architecture-overview.md)** - High-level system design, architecture diagrams, and component relationships
- **[02. Infrastructure Components](./docs/02-infrastructure-components.md)** - Detailed breakdown of all AWS resources (EKS, VPC, MSK, etc.)
- **[03. Network Architecture](./docs/03-network-architecture.md)** - VPC design, subnets, security groups, routing, and connectivity

### Security & Access
- **[04. Security Model](./docs/04-security-model.md)** - Comprehensive security strategy covering network, IAM, data, and access control
- **[05. IAM and RBAC](./docs/05-iam-rbac.md)** - Identity management, IRSA, OIDC integration, and Kubernetes RBAC

### Application Layer
- **[06. Application Architecture](./docs/06-application-architecture.md)** - Microservices design, NestJS patterns, Kafka integration
- **[07. Kafka Architecture](./docs/07-kafka-architecture.md)** - Event streaming design, topics, partitions, consumer groups

### Deployment & Automation
- **[08. Terraform Structure](./docs/08-terraform-structure.md)** - IaC organization, modules, state management, and best practices
- **[09. Kubernetes Manifests](./docs/09-kubernetes-manifests.md)** - Deployment patterns, Helm vs Kustomize, service configurations
- **[10. CI/CD Pipeline](./docs/10-cicd-pipeline.md)** - GitHub Actions workflows, release flows, and deployment strategies
- **[11. Container Registry](./docs/11-container-registry.md)** - ECR setup, image scanning, lifecycle policies

### Observability & Monitoring
- **[12. Observability Stack](./docs/12-observability-stack.md)** - Prometheus, Loki, Grafana setup and integration
- **[13. Monitoring Dashboards](./docs/13-monitoring-dashboards.md)** - Pre-built dashboards and alerting rules
- **[14. Logging Strategy](./docs/14-logging-strategy.md)** - Log collection, aggregation, retention, and analysis

### Scaling & Performance
- **[15. Autoscaling Strategy](./docs/15-autoscaling-strategy.md)** - HPA, VPA, Cluster Autoscaler, and Karpenter
- **[16. Performance Optimization](./docs/16-performance-optimization.md)** - Best practices for optimal resource utilization

### Operations
- **[17. Operations Guide](./docs/17-operations-guide.md)** - Day-to-day operations, maintenance, and troubleshooting
- **[18. Disaster Recovery](./docs/18-disaster-recovery.md)** - Backup strategies, RTO/RPO, recovery procedures
- **[19. Cost Analysis](./docs/19-cost-analysis.md)** - Detailed cost breakdown and optimization strategies

### Implementation
- **[20. Step-by-Step Implementation](./docs/20-step-by-step-implementation.md)** - Complete beginner-friendly guide for setting up the entire system
- **[21. Bastion Host Setup](./docs/21-bastion-host-setup.md)** - Secure access configuration and tools installation
- **[22. Troubleshooting Guide](./docs/22-troubleshooting-guide.md)** - Common issues and resolution procedures

## 🚀 Quick Start

### Prerequisites

- AWS Account with appropriate permissions
- AWS CLI configured
- Terraform >= 1.5.0
- kubectl >= 1.27
- Docker Desktop or similar
- GitHub account with Actions enabled
- Basic understanding of Kubernetes, Docker, and AWS

### Getting Started

1. **Start Here**: Read the [System Architecture Overview](./docs/01-architecture-overview.md) to understand the big picture
2. **Plan Your Infrastructure**: Review [Infrastructure Components](./docs/02-infrastructure-components.md) and [Cost Analysis](./docs/19-cost-analysis.md)
3. **Follow Implementation**: Use the [Step-by-Step Implementation Guide](./docs/20-step-by-step-implementation.md) to build your infrastructure
4. **Set Up CI/CD**: Configure [GitHub Actions pipelines](./docs/10-cicd-pipeline.md)
5. **Deploy Applications**: Follow [Application Architecture](./docs/06-application-architecture.md) and [Kubernetes Manifests](./docs/09-kubernetes-manifests.md)
6. **Enable Monitoring**: Set up [Observability Stack](./docs/12-observability-stack.md)
7. **Operational Readiness**: Review [Operations Guide](./docs/17-operations-guide.md) and [Disaster Recovery](./docs/18-disaster-recovery.md)

## 📊 Architecture Highlights

### Multi-Environment Setup
```
Production Environment          Development Environment
├── EKS Cluster (prod)         ├── EKS Cluster (dev)
├── MSK Kafka (prod)           ├── MSK Kafka (dev)
├── RDS Aurora (prod)          ├── RDS PostgreSQL (dev)
└── Separate VPC               └── Separate VPC
```

### CI/CD Flow
```
GitHub Push → Build & Test → ECR Push → Dev Auto-Deploy → Manual Approval → Prod Deploy
```

### Observability
```
Application Metrics → Prometheus → Grafana Dashboards
Application Logs → Loki → Grafana Log Viewer
Kafka Metrics → JMX Exporter → Prometheus → Grafana
```

## 💰 Cost Estimation (Monthly)

| Component | Estimated Cost |
|-----------|----------------|
| EKS Control Plane (2 clusters) | ~$144 |
| EC2 Worker Nodes (mixed spot/on-demand) | ~$400-800 |
| MSK Kafka | ~$400-600 |
| RDS/Aurora | ~$100-300 |
| NAT Gateway | ~$96 |
| Load Balancers | ~$50 |
| ECR, S3, CloudWatch | ~$50-100 |
| **Total Estimated** | **~$1,240-2,090/month** |

*Note: Costs vary based on usage, instance types, and optimization strategies. See [Cost Analysis](./docs/19-cost-analysis.md) for detailed breakdown.*

## 🎓 Learning Path

### For DevOps Beginners
1. Start with architecture diagrams to understand components
2. Follow the step-by-step guide in order
3. Set up dev environment first
4. Experiment with manual deployments before automating
5. Learn monitoring before going to production

### For Experienced Engineers
1. Review architecture decisions and tradeoffs
2. Adapt Terraform modules to your needs
3. Customize CI/CD pipelines for your workflow
4. Implement advanced features (GitOps, service mesh)
5. Optimize costs based on your usage patterns

## 🔧 Technology Stack

- **Cloud Platform**: AWS
- **Container Orchestration**: Kubernetes (EKS)
- **Infrastructure as Code**: Terraform
- **CI/CD**: GitHub Actions
- **Container Registry**: Amazon ECR
- **Event Streaming**: Apache Kafka (Amazon MSK or Strimzi)
- **Application Runtime**: Node.js (NestJS)
- **Monitoring**: Prometheus + Grafana
- **Logging**: Loki + Promtail
- **Ingress**: AWS Load Balancer Controller
- **DNS**: Route53
- **Secrets Management**: AWS Secrets Manager / External Secrets Operator

## 📖 Best Practices Covered

- ✅ Infrastructure as Code (IaC) with Terraform
- ✅ GitOps principles with GitHub Actions
- ✅ Immutable infrastructure
- ✅ Zero-downtime deployments
- ✅ Horizontal and vertical autoscaling
- ✅ Multi-AZ high availability
- ✅ Comprehensive observability
- ✅ Security in depth (network, IAM, data)
- ✅ Cost optimization strategies
- ✅ Disaster recovery planning
- ✅ Environment separation
- ✅ Secret management
- ✅ Container image scanning
- ✅ Pod-level IAM (IRSA)
- ✅ Network policies and segmentation

## 🤝 Contributing

This is a living document. As you implement this infrastructure and learn from it, consider:
- Documenting lessons learned
- Adding troubleshooting tips
- Improving automation scripts
- Sharing optimization strategies

## 📄 License

This project documentation is provided as-is for educational and implementation purposes.

## 🔗 Additional Resources

- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [NestJS Documentation](https://docs.nestjs.com/)

---

**Ready to build enterprise-grade infrastructure?** Start with [Architecture Overview](./docs/01-architecture-overview.md) →
