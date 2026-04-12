# GURU: Multi-Region Active-Active Kubernetes

**Track:** Platform Engineering | **Level:** Guru (3+ years) | **Duration:** 6–8 weeks

---

## Overview

### Project: Production-Grade Multi-Region AKS/EKS Platform

### What You'll Build

- Two Kubernetes clusters in different cloud regions (e.g., East US + West Europe, or us-east-1 + eu-west-1)
- Global traffic routing with intelligent failover using Azure Front Door or AWS Global Accelerator
- Cluster state synchronised via ArgoCD ApplicationSets targeting both clusters
- Cross-region data replication strategy with defined RTO/RPO targets
- A complete Disaster Recovery runbook with step-by-step failover and failback procedures
- Chaos engineering validation of the failover path

### Learning Outcomes

- Design for regional blast radius — isolate failures so one region never kills both
- Engineer traffic shifting at the global DNS/anycast layer
- Define and enforce RTO and RPO targets with runbook-backed procedures
- Manage multi-cluster GitOps with ArgoCD cluster registration
- Validate DR procedures before you need them

### Tech Stack

- **Cloud:** Azure (AKS) or AWS (EKS) — examples below use Azure, with AWS equivalents noted
- **IaC:** Terraform
- **Orchestration:** Kubernetes, Helm
- **GitOps:** ArgoCD with ApplicationSets
- **Traffic:** Azure Front Door (or AWS Global Accelerator)
- **Backup/DR:** Velero
- **Observability:** Prometheus, Grafana, Alertmanager
- **Chaos:** Chaos Mesh

---

## Architecture

```
                    ┌──────────────────────────────────┐
                    │   Azure Front Door / Global LB    │
                    │   health-probe every 30s          │
                    │   latency-based routing           │
                    └─────────────┬────────────────────┘
                                  │
              ┌───────────────────┴──────────────────────┐
              │                                           │
              ▼                                           ▼
  ┌───────────────────────┐               ┌───────────────────────┐
  │   Region A: East US   │               │  Region B: West EU    │
  │   AKS Cluster A       │               │  AKS Cluster B        │
  │                       │               │                       │
  │  ┌─────────────────┐  │               │  ┌─────────────────┐  │
  │  │ Ingress / AGIC  │  │               │  │ Ingress / AGIC  │  │
  │  └────────┬────────┘  │               │  └────────┬────────┘  │
  │           │            │               │           │            │
  │  ┌────────▼────────┐  │               │  ┌────────▼────────┐  │
  │  │  App Workloads  │  │               │  │  App Workloads  │  │
  │  │  (replicated)   │  │               │  │  (replicated)   │  │
  │  └────────┬────────┘  │               │  └────────┬────────┘  │
  │           │            │               │           │            │
  │  ┌────────▼────────┐  │               │  ┌────────▼────────┐  │
  │  │  Azure DB       │◄─┼───────────────┼─►│  Azure DB       │  │
  │  │  (geo-replica)  │  │  async repl   │  │  (geo-replica)  │  │
  │  └─────────────────┘  │               │  └─────────────────┘  │
  └───────────────────────┘               └───────────────────────┘
              ▲                                           ▲
              └──────────────────┬────────────────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │   ArgoCD (Hub cluster or    │
                    │   Cluster A acts as hub)    │
                    │   ApplicationSet targets    │
                    │   both clusters             │
                    └─────────────────────────────┘
                                  ▲
                                  │ GitOps
                    ┌─────────────┴──────────────┐
                    │   gitops-repo (GitHub)      │
                    └─────────────────────────────┘
```

### Design Principles

| Principle | Implementation |
|---|---|
| Regional isolation | Each cluster is fully self-sufficient — no runtime dependency on the other region |
| No shared state at runtime | Database geo-replication runs async; app reads from local replica |
| Traffic fails over in < 60s | Front Door health probes detect failure and reroute within one probe cycle |
| Config is symmetric | Same Helm values, same ArgoCD ApplicationSet, same Terraform module for both clusters |
| DR is tested | Chaos Mesh and quarterly game days validate every runbook step before it's needed |

---

## RTO / RPO Targets

| Environment | RTO | RPO | Notes |
|---|---|---|---|
| Production | < 5 minutes | < 30 seconds | Front Door failover + async DB replication lag |
| Staging | < 15 minutes | < 5 minutes | Lower priority, manual failover acceptable |
| Dev | Best effort | N/A | No DR requirement |

---

## Terraform Layout

```
infra/
├── modules/
│   ├── aks-cluster/           ← reusable AKS module (used by both regions)
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── networking/            ← VNet, subnets, NSGs
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── front-door/            ← Azure Front Door global routing
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── region-a/              ← East US
│   │   ├── main.tf            ← calls modules, region-specific vars
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf         ← remote state: storage account region-a
│   ├── region-b/              ← West Europe
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf         ← remote state: storage account region-b
│   └── global/                ← Front Door, DNS, shared resources
│       ├── main.tf
│       ├── variables.tf
│       └── backend.tf
└── scripts/
    ├── bootstrap.sh           ← provision both regions in order
    └── destroy.sh
```

### Module: `modules/aks-cluster/main.tf`

```hcl
# Reusable AKS cluster module
# Called identically by region-a and region-b with different variable values

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

resource "azurerm_resource_group" "main" {
  name     = "rg-${var.cluster_name}-${var.environment}"
  location = var.location
  tags     = var.tags
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.cluster_name}"
  address_space       = [var.vnet_cidr]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  tags                = var.tags
}

resource "azurerm_subnet" "aks" {
  name                 = "snet-aks"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.aks_subnet_cidr]
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = var.cluster_name
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = var.cluster_name
  kubernetes_version  = var.kubernetes_version

  # Node pool configuration
  default_node_pool {
    name                = "system"
    node_count          = var.system_node_count
    vm_size             = var.system_vm_size
    vnet_subnet_id      = azurerm_subnet.aks.id
    os_disk_size_gb     = 128
    type                = "VirtualMachineScaleSets"

    upgrade_settings {
      max_surge = "33%"
    }
  }

  # Managed identity — no service principal credentials to rotate
  identity {
    type = "SystemAssigned"
  }

  # Networking
  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
    service_cidr      = var.service_cidr
    dns_service_ip    = var.dns_service_ip
  }

  # Enable Azure Monitor integration
  monitor_metrics {
    annotations_allowed = null
    labels_allowed      = null
  }

  oms_agent {
    log_analytics_workspace_id = var.log_analytics_workspace_id
  }

  tags = var.tags
}

# User node pool for application workloads (separate from system pool)
resource "azurerm_kubernetes_cluster_node_pool" "apps" {
  name                  = "apps"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = var.app_vm_size
  node_count            = var.app_node_count
  vnet_subnet_id        = azurerm_subnet.aks.id

  node_labels = {
    "workload" = "application"
    "region"   = var.location
  }

  tags = var.tags
}
```

### `modules/aks-cluster/variables.tf`

```hcl
variable "cluster_name"              { type = string }
variable "environment"               { type = string }
variable "location"                  { type = string }
variable "kubernetes_version"        { type = string  default = "1.28" }
variable "vnet_cidr"                 { type = string }
variable "aks_subnet_cidr"           { type = string }
variable "service_cidr"              { type = string  default = "10.0.0.0/16" }
variable "dns_service_ip"            { type = string  default = "10.0.0.10" }
variable "system_node_count"         { type = number  default = 2 }
variable "system_vm_size"            { type = string  default = "Standard_D2s_v3" }
variable "app_node_count"            { type = number  default = 3 }
variable "app_vm_size"               { type = string  default = "Standard_D4s_v3" }
variable "log_analytics_workspace_id"{ type = string }
variable "tags"                      { type = map(string) default = {} }
```

### `environments/region-a/main.tf`

```hcl
# Region A — East US
# Uses the same module as region-b, different variable values only

module "networking" {
  source   = "../../modules/networking"
  location = "eastus"
  name     = "eastus"
  tags     = local.tags
}

module "aks" {
  source = "../../modules/aks-cluster"

  cluster_name               = "aks-prod-eastus"
  environment                = "prod"
  location                   = "eastus"
  kubernetes_version         = "1.28"
  vnet_cidr                  = "10.1.0.0/16"
  aks_subnet_cidr            = "10.1.1.0/24"
  service_cidr               = "10.100.0.0/16"
  dns_service_ip             = "10.100.0.10"
  system_node_count          = 2
  app_node_count             = 3
  app_vm_size                = "Standard_D4s_v3"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  tags                       = local.tags
}
```

### `environments/region-b/main.tf`

```hcl
# Region B — West Europe
# Identical module call — only location, cluster_name, and CIDR ranges differ

module "aks" {
  source = "../../modules/aks-cluster"

  cluster_name               = "aks-prod-westeu"
  environment                = "prod"
  location                   = "westeurope"
  kubernetes_version         = "1.28"
  vnet_cidr                  = "10.2.0.0/16"     # non-overlapping with region-a
  aks_subnet_cidr            = "10.2.1.0/24"
  service_cidr               = "10.200.0.0/16"
  dns_service_ip             = "10.200.0.10"
  system_node_count          = 2
  app_node_count             = 3
  app_vm_size                = "Standard_D4s_v3"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  tags                       = local.tags
}
```

### `environments/global/main.tf` — Azure Front Door

```hcl
# Global traffic routing with health-based failover

resource "azurerm_cdn_frontdoor_profile" "main" {
  name                = "fd-myapp-prod"
  resource_group_name = azurerm_resource_group.global.name
  sku_name            = "Premium_AzureFrontDoor"
  tags                = local.tags
}

resource "azurerm_cdn_frontdoor_origin_group" "main" {
  name                     = "og-myapp"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.main.id

  load_balancing {
    additional_latency_in_milliseconds = 50   # prefer closer region within 50ms
    sample_size                        = 4
    successful_samples_required        = 3
  }

  health_probe {
    path                = "/healthz"
    protocol            = "Https"
    interval_in_seconds = 30    # check every 30 seconds
    request_type        = "GET"
  }
}

resource "azurerm_cdn_frontdoor_origin" "region_a" {
  name                          = "origin-eastus"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.main.id
  enabled                       = true

  host_name          = var.region_a_ingress_ip
  http_port          = 80
  https_port         = 443
  origin_host_header = var.app_hostname
  priority           = 1        # primary region
  weight             = 1000
}

resource "azurerm_cdn_frontdoor_origin" "region_b" {
  name                          = "origin-westeu"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.main.id
  enabled                       = true

  host_name          = var.region_b_ingress_ip
  http_port          = 80
  https_port         = 443
  origin_host_header = var.app_hostname
  priority           = 2        # failover region — used only if region-a unhealthy
  weight             = 1000
}
```

---

## ArgoCD Multi-Cluster ApplicationSet

### Step 1: Register both clusters in ArgoCD

```bash
# Assuming ArgoCD runs in cluster-a (hub cluster)

# Add credentials for cluster-a (in-cluster)
# (Already available — ArgoCD runs here)

# Add credentials for cluster-b (remote)
argocd cluster add aks-prod-westeu \
  --kubeconfig ~/.kube/config-westeu \
  --name cluster-westeu \
  --in-cluster=false

# Verify both clusters are registered
argocd cluster list
```

### Step 2: Label clusters for ApplicationSet targeting

```bash
# Apply labels so ApplicationSet can target by region
kubectl label secret -n argocd \
  $(argocd cluster list -o json | jq -r '.[] | select(.name=="cluster-eastus") | .name') \
  region=eastus environment=prod

kubectl label secret -n argocd \
  $(argocd cluster list -o json | jq -r '.[] | select(.name=="cluster-westeu") | .name') \
  region=westeu environment=prod
```

### `applicationsets/myapp-multiregion.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-multiregion
  namespace: argocd
spec:
  generators:
    # Cluster generator: target clusters matching labels
    - clusters:
        selector:
          matchLabels:
            environment: prod

  template:
    metadata:
      name: "myapp-{{name}}"    # name = the ArgoCD cluster name
      labels:
        app: myapp
        region: "{{metadata.labels.region}}"
      annotations:
        # Link to runbook from ArgoCD UI
        notifications.argoproj.io/subscribe.on-sync-failed.slack: ops-alerts
    spec:
      project: production

      source:
        repoURL: https://github.com/YOUR_ORG/gitops-repo.git
        targetRevision: HEAD
        path: apps/myapp/base
        helm:
          valueFiles:
            - "../prod/values.yaml"
          # Inject region as a dynamic value
          parameters:
            - name: region
              value: "{{metadata.labels.region}}"

      destination:
        server: "{{server}}"    # the cluster API URL from ArgoCD registration
        namespace: myapp-prod

      syncPolicy:
        automated:
          prune: false          # never auto-delete in prod
          selfHeal: false       # humans own prod reconciliation
        syncOptions:
          - CreateNamespace=true
          - RespectIgnoreDifferences=true
        retry:
          limit: 3
          backoff:
            duration: 60s
            factor: 2
            maxDuration: 5m

      # Ignore fields that change at runtime (replica count managed by HPA)
      ignoreDifferences:
        - group: apps
          kind: Deployment
          jsonPointers:
            - /spec/replicas
```

---

## Failover Logic

### How Front Door detects failure

1. Front Door sends a `GET /healthz` probe to both origins every 30 seconds
2. If an origin fails 3 consecutive probes, Front Door marks it `Unhealthy`
3. All traffic shifts to the healthy origin within the next probe cycle
4. **Total detection + reroute time: ~90 seconds worst case, typically ~60 seconds**

### Application health check endpoint

Every service must expose `/healthz`. This is non-negotiable.

```python
# Python / FastAPI example
@app.get("/healthz")
async def health_check():
    # Check database connectivity
    try:
        await db.execute("SELECT 1")
        db_ok = True
    except Exception:
        db_ok = False

    if not db_ok:
        # Return 503 — Front Door will treat this origin as unhealthy
        raise HTTPException(status_code=503, detail="database unreachable")

    return {
        "status": "ok",
        "region": os.getenv("REGION", "unknown"),
        "timestamp": datetime.utcnow().isoformat()
    }
```

```yaml
# Kubernetes liveness and readiness probes
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
```

### PodDisruptionBudget — prevent both regions going down during maintenance

```yaml
# Apply this to BOTH clusters
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: myapp-prod
spec:
  minAvailable: 2    # always keep at least 2 pods running during node drains
  selector:
    matchLabels:
      app: myapp
```

---

## Velero Backup and Restore

Velero takes snapshots of Kubernetes state (all objects + persistent volume data) and stores them in blob storage. This is your safety net if a full cluster is lost.

### Install Velero on both clusters

```bash
# Install Velero CLI
brew install velero

# Install Velero on cluster-a (East US)
velero install \
  --provider azure \
  --plugins velero/velero-plugin-for-microsoft-azure:v1.8.0 \
  --bucket velero-backups-eastus \
  --secret-file ./credentials-velero \
  --backup-location-config \
    resourceGroup=rg-velero,storageAccount=stveleroeastus,subscriptionId=$SUB_ID \
  --snapshot-location-config \
    apiTimeout=5m,resourceGroup=rg-velero,subscriptionId=$SUB_ID

# Repeat for cluster-b (West Europe) — pointing to a DIFFERENT storage account
velero install \
  --provider azure \
  --plugins velero/velero-plugin-for-microsoft-azure:v1.8.0 \
  --bucket velero-backups-westeu \
  --secret-file ./credentials-velero \
  --backup-location-config \
    resourceGroup=rg-velero,storageAccount=stvelerowesteu,subscriptionId=$SUB_ID
```

### Schedule automated backups

```bash
# Run a full backup every hour, keep 48 backups (2 days)
velero schedule create myapp-hourly \
  --schedule="0 * * * *" \
  --ttl 48h \
  --include-namespaces myapp-prod \
  --storage-location default

# Verify the schedule was created
velero schedule get
```

---

## Disaster Recovery Runbook

### Severity Definition

| Severity | Definition | Response |
|---|---|---|
| P1 — Full region loss | Entire region-a cluster unreachable | Immediate failover, on-call paged |
| P2 — Degraded region | Region-a serving errors > 5%, health probes failing | Accelerated failover evaluation |
| P3 — Partial failure | Single namespace or service down in region-a | Investigate, no failover yet |

---

### Runbook — P1: Full Region Failover

**When to use:** Region-A is completely unreachable. Front Door has already shifted traffic to Region-B. This runbook confirms the shift, validates Region-B health, and optionally deprovisions Region-A.

---

#### Phase 1: Detect (Target: 0–5 minutes)

```bash
# Step 1.1: Confirm Front Door has detected the failure
az afd origin show \
  --resource-group rg-global \
  --profile-name fd-myapp-prod \
  --origin-group-name og-myapp \
  --origin-name origin-eastus \
  --query healthProbeSettings

# Expected: origin marked Disabled or health check failures

# Step 1.2: Confirm traffic is flowing to Region-B only
az afd metric show \
  --resource-group rg-global \
  --profile-name fd-myapp-prod \
  --metric RequestCount \
  --interval PT1M

# Step 1.3: Verify Region-B is healthy and serving traffic
kubectl get pods -n myapp-prod --context=aks-prod-westeu
kubectl get svc -n myapp-prod --context=aks-prod-westeu

# Step 1.4: Run a manual health check against the global endpoint
curl -v https://myapp.example.com/healthz
# Expected: HTTP 200, "region": "westeu"
```

**Decision gate:** If Region-B confirms healthy and global health check returns 200 → Front Door failover has succeeded. Proceed to Phase 2 to notify and validate.

---

#### Phase 2: Notify and Validate (Target: 5–15 minutes)

```bash
# Step 2.1: Post to incident Slack channel
# Use your incident template:
# "P1 INCIDENT: Region-A (eastus) unreachable at {TIME}
#  Traffic automatically failed over to Region-B (westeu) via Front Door.
#  Region-B status: HEALTHY | Investigating root cause | Bridge: {LINK}"

# Step 2.2: Check database replication lag
# Azure SQL Geo-Replica — check lag on the West EU replica
az sql db show \
  --resource-group rg-prod-westeu \
  --server sql-prod-westeu \
  --name myapp-db \
  --query "replicationLinks"

# Acceptable lag: < 30 seconds (within our RPO)
# If lag > 5 minutes: flag for data reconciliation after recovery

# Step 2.3: Validate application functionality end-to-end
# Run smoke tests against production global endpoint
bash scripts/smoke-test.sh https://myapp.example.com

# Step 2.4: Confirm ArgoCD reflects Region-B as the active target
argocd app list
argocd app get myapp-aks-prod-westeu
```

---

#### Phase 3: Scale Up Region-B (Target: 15–30 minutes)

If Region-B is now serving 100% of traffic, it may need more capacity.

```bash
# Step 3.1: Scale up the application deployment in Region-B
kubectl scale deployment myapp \
  -n myapp-prod \
  --replicas=6 \
  --context=aks-prod-westeu

# Step 3.2: Scale up the node pool if pods are pending
az aks nodepool scale \
  --resource-group rg-prod-westeu \
  --cluster-name aks-prod-westeu \
  --name apps \
  --node-count 6

# Step 3.3: Confirm all pods are running
kubectl get pods -n myapp-prod --context=aks-prod-westeu -w

# Step 3.4: Check HPA status
kubectl get hpa -n myapp-prod --context=aks-prod-westeu
```

---

#### Phase 4: Region-A Recovery or Rebuild (Target: 30 min – 4 hours)

**Option A: Region-A recovers naturally (cloud provider incident resolves)**

```bash
# Step 4.1: Verify Region-A cluster is reachable
kubectl get nodes --context=aks-prod-eastus

# Step 4.2: Check ArgoCD sync status for Region-A
argocd app get myapp-aks-prod-eastus

# Step 4.3: Sync Region-A to current GitOps state
argocd app sync myapp-aks-prod-eastus

# Step 4.4: Verify Region-A pods are healthy
kubectl get pods -n myapp-prod --context=aks-prod-eastus

# Step 4.5: Re-enable Region-A origin in Front Door
az afd origin update \
  --resource-group rg-global \
  --profile-name fd-myapp-prod \
  --origin-group-name og-myapp \
  --origin-name origin-eastus \
  --enabled-state Enabled

# Step 4.6: Monitor traffic distribution for 15 minutes
# Front Door will gradually restore traffic to Region-A based on health probes
```

**Option B: Region-A must be fully rebuilt from Velero backup**

```bash
# Step 4.1: Create a new AKS cluster in East US (Terraform)
cd infra/environments/region-a
terraform apply -target=module.aks

# Step 4.2: Point kubectl to the new cluster
az aks get-credentials \
  --resource-group rg-aks-prod-eastus \
  --name aks-prod-eastus \
  --overwrite-existing

# Step 4.3: Install Velero on the new cluster
velero install \
  --provider azure \
  --plugins velero/velero-plugin-for-microsoft-azure:v1.8.0 \
  --bucket velero-backups-eastus \
  --secret-file ./credentials-velero \
  --backup-location-config \
    resourceGroup=rg-velero,storageAccount=stveleroeastus,subscriptionId=$SUB_ID \
  --use-existing-crds

# Step 4.4: List available backups
velero backup get

# Step 4.5: Restore from the most recent backup
velero restore create \
  --from-backup myapp-hourly-<TIMESTAMP> \
  --include-namespaces myapp-prod \
  --wait

# Step 4.6: Verify restore completed
velero restore describe myapp-hourly-<TIMESTAMP>-restore
kubectl get pods -n myapp-prod

# Step 4.7: Re-register cluster with ArgoCD
argocd cluster add aks-prod-eastus \
  --kubeconfig ~/.kube/config \
  --name cluster-eastus

# Step 4.8: Sync via ArgoCD to get latest state from Git
argocd app sync myapp-aks-prod-eastus

# Step 4.9: Re-enable in Front Door (only after health checks pass)
curl https://REGION-A-INGRESS-IP/healthz   # must return 200
az afd origin update \
  --resource-group rg-global \
  --profile-name fd-myapp-prod \
  --origin-group-name og-myapp \
  --origin-name origin-eastus \
  --enabled-state Enabled
```

---

#### Phase 5: Post-Incident

```bash
# Step 5.1: Scale Region-B back to normal capacity
kubectl scale deployment myapp -n myapp-prod --replicas=3 --context=aks-prod-westeu
az aks nodepool scale \
  --resource-group rg-prod-westeu \
  --cluster-name aks-prod-westeu \
  --name apps \
  --node-count 3

# Step 5.2: Verify traffic is balanced across both regions
az afd metric show \
  --resource-group rg-global \
  --profile-name fd-myapp-prod \
  --metric RequestCount \
  --interval PT5M

# Step 5.3: Write the incident postmortem within 48 hours
# Template: docs/postmortem-template.md
# Cover: timeline, root cause, impact, detection gap, action items

# Step 5.4: Schedule chaos engineering test of this failover path
# Run within 2 weeks to validate the runbook is still accurate
```

---

## Chaos Engineering Validation

Run these experiments in staging before a production game day.

### Install Chaos Mesh

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  -n chaos-testing --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

### Experiment 1: Kill all pods in a namespace (simulates app crash)

```yaml
# chaos-pod-kill.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-myapp-pods
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: all
  selector:
    namespaces:
      - myapp-prod
    labelSelectors:
      app: myapp
  duration: "2m"
```

```bash
kubectl apply -f chaos-pod-kill.yaml
# Expected: Front Door detects /healthz failures within 90s
# Expected: Traffic shifts to Region-B automatically
# Expected: ArgoCD self-heal restarts pods in Region-A (if selfHeal enabled)
```

### Experiment 2: Network latency (simulates degraded region)

```yaml
# chaos-network-latency.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: high-latency-eastus
  namespace: chaos-testing
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - myapp-prod
  delay:
    latency: "500ms"
    correlation: "100"
    jitter: "0ms"
  duration: "5m"
  direction: to
```

### Game Day Checklist

Run quarterly. Document results in `docs/game-days/`.

```
[ ] Announce game day window (1 week in advance)
[ ] Confirm staging environment is production-identical
[ ] Baseline: record p99 latency and error rate before injection
[ ] Execute Experiment 1 (pod kill)
    [ ] Measure time to Front Door failover
    [ ] Confirm traffic is 100% on Region-B within SLA
[ ] Execute Experiment 2 (network latency)
    [ ] Confirm latency-based routing reduces load on degraded region
[ ] Execute Experiment 3 (full namespace deletion)
    [ ] Run velero restore
    [ ] Measure actual RTO vs target RTO
[ ] Restore all systems
[ ] Compare actual RTO/RPO vs targets
[ ] File action items for any gaps
[ ] Schedule next game day
```

---

## Observability

### SLO Alert — Regional traffic imbalance

```yaml
# alertmanager rule: fires if Region-B is receiving > 80% of traffic
# (indicates Region-A may be degraded but Front Door hasn't fully failed over)
groups:
  - name: multiregion.slos
    rules:
      - alert: RegionATrafficImbalance
        expr: |
          (
            sum(rate(http_requests_total{region="westeu"}[5m])) /
            sum(rate(http_requests_total[5m]))
          ) > 0.80
        for: 2m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Region-B receiving > 80% of traffic"
          description: "Region-A may be degraded. Check Front Door health probes."
          runbook: "https://wiki.example.com/runbooks/multiregion-failover"

      - alert: RegionHealthCheckFailing
        expr: probe_success{job="blackbox", instance=~".*eastus.*"} == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Region-A health checks failing"
          description: "Immediately check Front Door and cluster status in East US."
          runbook: "https://wiki.example.com/runbooks/multiregion-failover"
```

---

## Deliverables

| Deliverable | Description |
|---|---|
| Terraform modules | Reusable `aks-cluster`, `networking`, `front-door` modules |
| Two live clusters | East US and West Europe, provisioned symmetrically |
| ArgoCD ApplicationSet | Single YAML managing both clusters from GitOps repo |
| Azure Front Door config | Health-based failover with < 90s detection |
| Velero backup schedules | Hourly backups on both clusters, stored in separate accounts |
| DR Runbook | This document — validated in a game day |
| Chaos Mesh experiments | Pod-kill and network-latency experiments for quarterly game days |
| SLO alerts | Prometheus rules monitoring traffic imbalance and health check status |
| Game Day report | Post-exercise write-up comparing actual vs target RTO/RPO |

---

## Key Concepts You Now Understand

| Concept | What It Means |
|---|---|
| Blast radius | The scope of impact if one region fails — active-active limits it to < 100% |
| Active-active | Both regions serve real traffic simultaneously — neither is idle standby |
| RTO | Recovery Time Objective — maximum acceptable downtime during an incident |
| RPO | Recovery Point Objective — maximum acceptable data loss (measured in time) |
| Front Door health probes | Automated checks that drive traffic shifting decisions |
| Velero restore | Rebuilding cluster state from a point-in-time backup |
| Chaos engineering | Deliberately breaking things in a controlled way to find gaps before they find you |
| Game day | A structured exercise where the team practices executing the DR runbook |

---

## What to Explore Next

- Implement **Argo Rollouts** for canary and blue/green deployments within each region
- Add **cert-manager** for automated TLS certificate rotation across both clusters
- Build the **Internal Developer Platform** (Guru project 2) on top of this foundation
- Integrate **OPA Gatekeeper** policies to enforce security standards uniformly across both clusters
- Publish a **FinOps dashboard** tracking per-region cloud spend against a budget
