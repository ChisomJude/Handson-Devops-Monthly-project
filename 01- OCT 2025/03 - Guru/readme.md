# Super Level:  Resilient & Automated Secret Management with GitOps

 *Project: Vault on K8s (HA) with Helm, KMS auto-unseal, GitOps & DR (inspired by Rahman Badru - Interswitch)*

This project challenges you to deploy HashiCorp Vault, the industry-standard secrets manager, on Kubernetes in a highly available (HA) setup. The goal is to make Vault resilient, secure, and production-like by combining HA clustering, cloud KMS auto-unseal, GitOps configuration management, observability(optional), and disaster recovery automation.

**Goals**

- Run Vault in HA (Raft) on K8s.
- Auto-unseal with a cloud KMS (AWS KMS suggested).
- Enable Kubernetes auth, app roles, policies, and dynamic secrets.
- Ship metrics/logs to Prometheus/Grafana/Loki. (optional)
- Implement automated snapshots to S3 + restore drill.
- (Optional) Manage via ArgoCD (GitOps).



## Target Architecture (high-level)

- EKS/K8s cluster (3 worker nodes).
- Vault (Helm) → 3 replicas, Integrated Storage (Raft), PodDisruptionBudget, anti-affinity.
- Auto-unseal: AWS KMS + IAM.
- Ingress (NGINX) + TLS (cert-manager).
- Observability: Prometheus scraper + Grafana dashboards, Loki for logs.(optional)
- DR: vault operator snapshot to S3 on schedule (CronJob).
- App Namespace uses Kubernetes Auth + KV v2 & DB secrets engine.


Project Inspo for Next Month - Please fill this form - https://forms.gle/1LvhxHGKWmk9jXSy7
