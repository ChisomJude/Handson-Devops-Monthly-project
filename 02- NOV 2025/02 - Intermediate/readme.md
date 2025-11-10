#  INTERMEDIATE LEVEL: Containerized Application with CI/CD

**Goal:** Deploy a containerized application with automated pipelines and configuration management

---

## Project: Microservices Blog Platform

### What You'll Build:

- **Dockerized Node.js/Python application** (preferably create your own blog app using Python, or use this repo: [https://github.com/jimohabdol/week-6-assessment](https://github.com/jimohabdol/week-6-assessment))
- ECS/EKS cluster provisioned with Terraform
- CI/CD pipeline with GitHub Actions
- Configuration management with Ansible
- Application Load Balancer with SSL
- RDS database backend
- Monitoring with CloudWatch

---

## Learning Outcomes:

- Container orchestration (ECS or EKS basics)
- Advanced Terraform (modules, remote state, workspaces)
- CI/CD pipeline implementation
- Configuration management with Ansible
- Secrets management
- Load balancing and SSL/TLS
- Database integration
- Basic monitoring and logging

---

## Tech Stack:

- **Terraform** (with modules)
- **Docker & Docker Compose**
- **AWS** (ECS/EKS, RDS, ALB, ECR)
- **GitHub Actions**
- **Ansible**
- **Bash scripting**
- **Nginx/Application server**

---

## Implementation Steps:

### Week 1:
- Dockerize application
- Create Terraform modules (VPC, ECS, RDS)
- Setup remote state in S3

### Week 2:
- Build ECS cluster with Terraform
- Implement service discovery
- Configure Application Load Balancer

### Week 3:
- Create CI/CD pipeline in GitHub Actions
- Automated testing and Docker image builds
- Deploy to ECR and ECS

### Week 4:
- Ansible playbooks for server configuration
- Implement secrets management (AWS Secrets Manager)
- Setup CloudWatch dashboards and alarms

---

## Deliverables:

- Complete Terraform module structure
- Dockerized application
- GitHub Actions workflows
- Ansible playbooks
- Monitoring dashboards
- Comprehensive documentation

---

## Key Files Structure:
```
intermediate-project/
├── terraform/
│   ├── modules/
│   │   ├── vpc/
│   │   ├── ecs/
│   │   ├── rds/
│   │   └── alb/
│   ├── environments/
│   │   ├── dev/
│   │   └── prod/
│   └── backend.tf
├── app/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .github/
│   └── workflows/
│       └── deploy.yml
├── ansible/
│   ├── playbooks/
│   └── inventory/
└── scripts/
    └── deploy.sh
```

---

## 📊 Progression Path

| Skill Area | Focus |
|------------|-------|
| **Containers** | Docker, ECS orchestration |
| **IaC** | Terraform modules, remote state |
| **CI/CD** | GitHub Actions pipelines |
| **Configuration** | Ansible playbooks |
| **Security** | Secrets management, SSL |
| **Monitoring** | CloudWatch dashboards |




---

## 💡 Success Criteria

- ✅ Application runs in containers on ECS/EKS
- ✅ Fully automated CI/CD pipeline
- ✅ Infrastructure defined as code
- ✅ Secure secrets management
- ✅ SSL/TLS encryption enabled
- ✅ Database properly integrated
- ✅ Monitoring and alerting configured
- ✅ Complete documentation

---

## Additional Resources:

- [Docker Documentation](https://docs.docker.com/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Ansible Documentation](https://docs.ansible.com/)
- [AWS ECS Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/intro.html)