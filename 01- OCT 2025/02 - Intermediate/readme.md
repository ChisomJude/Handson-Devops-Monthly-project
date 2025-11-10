# Intermediate Level: Automation & Infrastructure as Code

## Project: Automated Multi-Environment Deployment with Terraform & CI/CD

*This project focuses on building an Infrastructure as Code (IaC) workflow using Terraform and combining it with a CI/CD pipeline (e.g., GitHub Actions) to deploy infrastructure across multiple environments (Dev, Staging, and Production). The challenge is not just provisioning resources, but doing it in a repeatable, automated, and environment-isolated way.*


### Tasks:


- Write Terraform to provision Dev, Staging, and Prod environments (VPCs, EC2/ECS, IAM).
- Implement remote backend & state locking (S3 + DynamoDB).
- Create a GitHub Actions pipeline that deploys to the right environment based on branch/tag.
  - dev branch → Dev environment
  - staging branch → Staging
  - master branch → Production

- Add a rollback mechanism in the pipeline (deploy the last known stable version).
- Deliverable: IaC repo + CI/CD pipeline + environment switch.
