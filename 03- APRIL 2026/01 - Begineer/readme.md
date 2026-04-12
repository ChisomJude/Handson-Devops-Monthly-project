# INTERMEDIATE: GitOps Deployment Pipeline with ArgoCD

**Track:** Platform Engineering | **Level:** Intermediate (1–3 years) | **Duration:** 4 weeks

---

## Overview

### Project: GitOps Multi-Environment Promotion Pipeline

### What You'll Build

- A GitOps mono-repo with a clear structure separating application code from Kubernetes manifests
- ArgoCD installed on a local or cloud Kubernetes cluster
- Helm charts for a sample web application
- An ArgoCD **ApplicationSet** that automatically manages dev, staging, and production environments
- A promotion strategy: code flows from dev → staging → prod through Git — no manual `kubectl apply` ever

### Learning Outcomes

- Understand GitOps: why Git is the single source of truth for infrastructure state
- Write and manage Helm charts for Kubernetes applications
- Use ArgoCD ApplicationSets to manage multiple environments from one definition
- Implement safe promotion strategies with sync policies and rollback
- Understand how to detect and resolve ArgoCD drift

### Tech Stack

- Kubernetes (K3s locally, or AKS/EKS/GKE for cloud)
- ArgoCD
- Helm
- GitHub Actions (CI — image build and push)
- Docker + DockerHub or GitHub Container Registry
- `kubectl`, `argocd` CLI

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     GitHub (Source of Truth)             │
│                                                          │
│  app-repo/                  gitops-repo/                 │
│  ├── src/                   ├── apps/                    │
│  ├── Dockerfile             │   └── myapp/               │
│  └── .github/               │       ├── dev/             │
│      └── workflows/         │       ├── staging/         │
│          └── ci.yaml        │       └── prod/            │
│               │             └── applicationsets/         │
│               │                 └── myapp-appset.yaml    │
│               ▼                         │                │
│          Build & push image             │                │
│          Update image tag               ▼                │
│          in gitops-repo    ─────────────────────────     │
└─────────────────────────────────────────────────────────┘
                                          │
                                          ▼
                               ┌──────────────────┐
                               │   ArgoCD          │
                               │  (watches repo)   │
                               └──────────────────┘
                                    │       │
                          ┌─────────┘       └──────────┐
                          ▼                             ▼
                   ┌────────────┐              ┌──────────────┐
                   │  dev NS    │              │  staging NS  │
                   │ (auto-sync)│              │ (auto-sync)  │
                   └────────────┘              └──────────────┘
                                                      │
                                          manual gate │
                                                      ▼
                                              ┌────────────┐
                                              │  prod NS   │
                                              │ (manual    │
                                              │  sync)     │
                                              └────────────┘
```

---

## Repository Layout

You will maintain **two** repositories:

### 1. `app-repo` — Application source code

```
app-repo/
├── src/
│   └── main.py               ← your application code
├── Dockerfile
├── .github/
│   └── workflows/
│       └── ci.yaml           ← builds image, pushes to registry,
│                               updates image tag in gitops-repo
└── README.md
```

### 2. `gitops-repo` — Kubernetes manifests and ArgoCD configs

```
gitops-repo/
├── apps/
│   └── myapp/
│       ├── base/                      ← Helm chart (shared across envs)
│       │   ├── Chart.yaml
│       │   ├── values.yaml            ← default values
│       │   └── templates/
│       │       ├── deployment.yaml
│       │       ├── service.yaml
│       │       ├── hpa.yaml
│       │       └── ingress.yaml
│       ├── dev/
│       │   └── values.yaml            ← dev overrides
│       ├── staging/
│       │   └── values.yaml            ← staging overrides
│       └── prod/
│           └── values.yaml            ← prod overrides
├── applicationsets/
│   └── myapp-appset.yaml              ← THE file that manages all 3 envs
├── argocd/
│   └── argocd-install.yaml            ← ArgoCD bootstrap (optional)
└── README.md
```

---

## Implementation Steps

### Week 1 — Install Kubernetes and ArgoCD

**Step 1: Get a Kubernetes cluster**

Locally with K3s (fastest option):

```bash
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes
```

Or use an existing cloud cluster. Confirm it is running:

```bash
kubectl cluster-info
```

**Step 2: Install ArgoCD**

```bash
# Create a dedicated namespace for ArgoCD
kubectl create namespace argocd

# Install ArgoCD from the official manifest
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD pods to be ready (takes 1-2 minutes)
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=120s

# Check all pods are Running
kubectl get pods -n argocd
```

**Step 3: Access the ArgoCD UI**

```bash
# Forward port to your laptop
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Open `https://localhost:8080` — login with `admin` and the password above.

**Step 4: Log in via CLI**

```bash
# Install argocd CLI
brew install argocd                     # macOS
# or: curl -sSL -o /usr/local/bin/argocd https://... (Linux)

argocd login localhost:8080 \
  --username admin \
  --password <YOUR_PASSWORD> \
  --insecure
```

---

### Week 2 — Build the Helm Chart

**Step 1: Create the base Helm chart**

```bash
mkdir -p gitops-repo/apps/myapp/base/templates
cd gitops-repo/apps/myapp/base
```

**`Chart.yaml`:**

```yaml
apiVersion: v2
name: myapp
description: Sample web application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

**`values.yaml` (base defaults):**

```yaml
# Default values — overridden per environment
replicaCount: 1

image:
  repository: ghcr.io/YOUR_ORG/myapp
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  host: ""

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70

env: []
```

**`templates/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    app: {{ .Release.Name }}
    environment: {{ .Values.environment | default "unknown" }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: myapp
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            {{- toYaml .Values.env | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**`templates/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
```

**Step 2: Create per-environment value overrides**

**`apps/myapp/dev/values.yaml`:**

```yaml
# Dev: lightweight, fast iteration
replicaCount: 1
environment: dev

image:
  tag: "latest"   # always pull the newest build

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 100m
    memory: 128Mi

env:
  - name: APP_ENV
    value: development
  - name: LOG_LEVEL
    value: debug
```

**`apps/myapp/staging/values.yaml`:**

```yaml
# Staging: mirrors prod sizing, used for final validation
replicaCount: 2
environment: staging

image:
  tag: "stable"   # pinned to a validated tag

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

env:
  - name: APP_ENV
    value: staging
  - name: LOG_LEVEL
    value: info
```

**`apps/myapp/prod/values.yaml`:**

```yaml
# Prod: hardened, scaled, conservative
replicaCount: 3
environment: production

image:
  tag: "v1.0.0"   # always an explicit version tag — never "latest"

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  - name: APP_ENV
    value: production
  - name: LOG_LEVEL
    value: warn
```

---

### Week 3 — Write the ApplicationSet

An ApplicationSet is a single ArgoCD resource that generates multiple Application objects — one per environment. Instead of creating three separate ArgoCD Applications by hand, you define the pattern once.

**`applicationsets/myapp-appset.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  # ─────────────────────────────────────────
  # Generator: defines the list of environments
  # Each item creates one ArgoCD Application
  # ─────────────────────────────────────────
  generators:
    - list:
        elements:
          - env: dev
            namespace: myapp-dev
            autoSync: "true"
            selfHeal: "true"
            valuesFile: dev/values.yaml

          - env: staging
            namespace: myapp-staging
            autoSync: "true"
            selfHeal: "true"
            valuesFile: staging/values.yaml

          - env: prod
            namespace: myapp-prod
            autoSync: "false"     # MANUAL sync for prod — humans decide
            selfHeal: "false"
            valuesFile: prod/values.yaml

  # ─────────────────────────────────────────
  # Template: the Application spec, parameterised
  # ─────────────────────────────────────────
  template:
    metadata:
      name: "myapp-{{env}}"
      labels:
        app: myapp
        environment: "{{env}}"
    spec:
      project: default

      # Where to find the Helm chart in Git
      source:
        repoURL: https://github.com/YOUR_ORG/gitops-repo.git
        targetRevision: HEAD
        path: apps/myapp/base
        helm:
          valueFiles:
            - "../{{valuesFile}}"    # load the env-specific values

      # Where to deploy in the cluster
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"

      # Sync policy controls how ArgoCD applies changes
      syncPolicy:
        automated:
          prune: "{{autoSync}}"      # delete resources removed from Git
          selfHeal: "{{selfHeal}}"   # re-apply if someone manually changes cluster state
        syncOptions:
          - CreateNamespace=true     # create the namespace if it doesn't exist
          - PrunePropagationPolicy=foreground
          - ApplyOutOfSyncOnly=true  # only sync resources that are actually out of sync

      # Retry on transient failures
      retry:
        limit: 3
        backoff:
          duration: 30s
          factor: 2
          maxDuration: 3m
```

**Apply the ApplicationSet:**

```bash
kubectl apply -f applicationsets/myapp-appset.yaml

# Watch ArgoCD create the three Applications
argocd app list

# Check sync status
argocd app get myapp-dev
argocd app get myapp-staging
argocd app get myapp-prod
```

---

### Week 4 — CI Pipeline and Promotion Strategy

**Step 1: GitHub Actions CI pipeline**

This workflow builds your Docker image, pushes it, and updates the image tag in the gitops-repo to trigger ArgoCD.

**`.github/workflows/ci.yaml` (in `app-repo`):**

```yaml
name: CI — Build and promote

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    name: Build and push image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}

  promote-to-dev:
    name: Update dev image tag in gitops-repo
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout gitops-repo
        uses: actions/checkout@v4
        with:
          repository: YOUR_ORG/gitops-repo
          token: ${{ secrets.GITOPS_TOKEN }}   # PAT with write access
          path: gitops-repo

      - name: Update dev image tag
        run: |
          IMAGE_TAG="sha-${{ github.sha }}"
          cd gitops-repo
          # Replace the tag in dev values file
          sed -i "s|tag:.*|tag: \"${IMAGE_TAG}\"|" \
            apps/myapp/dev/values.yaml

      - name: Commit and push
        run: |
          cd gitops-repo
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add apps/myapp/dev/values.yaml
          git commit -m "chore(dev): update myapp image to sha-${{ github.sha }}"
          git push
```

**Step 2: Promotion strategy — dev → staging → prod**

```
┌────────────┐    Auto    ┌─────────────┐   Manual PR  ┌──────────────┐   Manual sync  ┌──────────────┐
│  Code push │ ─────────▶ │  dev env    │ ────────────▶ │ staging env  │ ──────────────▶│  prod env    │
│  to main   │            │ auto-synced │              │  auto-synced │                │ manual only  │
└────────────┘            └─────────────┘              └──────────────┘                └──────────────┘
                          ArgoCD watches               Engineer updates                 Team reviews,
                          dev/values.yaml              staging/values.yaml              approves, then:
                          and applies changes           via PR + review                 argocd app sync
                                                                                        myapp-prod
```

**Promote from dev to staging (via PR):**

```bash
# In gitops-repo — update staging to use the same validated image tag
git checkout -b promote/myapp-to-staging

# Update staging values.yaml with the tested image tag
IMAGE_TAG="sha-abc1234"
sed -i "s|tag:.*|tag: \"${IMAGE_TAG}\"|" apps/myapp/staging/values.yaml

git add apps/myapp/staging/values.yaml
git commit -m "chore(staging): promote myapp ${IMAGE_TAG}"
git push origin promote/myapp-to-staging
# Open PR → get review → merge → ArgoCD auto-syncs staging
```

**Promote from staging to prod (manual ArgoCD sync):**

```bash
# Update prod values.yaml with an explicit version tag
git checkout -b promote/myapp-to-prod
sed -i "s|tag:.*|tag: \"v1.2.0\"|" apps/myapp/prod/values.yaml
git add apps/myapp/prod/values.yaml
git commit -m "chore(prod): release myapp v1.2.0"
git push origin promote/myapp-to-prod
# Open PR → senior review → merge

# After merge: a human triggers the prod sync
argocd app sync myapp-prod

# Monitor the rollout
argocd app get myapp-prod --watch
kubectl rollout status deployment/myapp -n myapp-prod
```

**Rollback prod if something goes wrong:**

```bash
# Option 1: Roll back via ArgoCD to a previous Git revision
argocd app rollback myapp-prod

# Option 2: Revert the commit in gitops-repo (preferred)
git revert HEAD
git push origin main
# ArgoCD will auto-sync back to the previous state

# Option 3: Emergency — force sync to a specific revision
argocd app sync myapp-prod --revision <COMMIT_SHA>
```

---

## Sync Policies Explained

| Policy | Dev | Staging | Prod | Why |
|---|---|---|---|---|
| `automated.prune` | true | true | false | Prod: never delete resources automatically |
| `automated.selfHeal` | true | true | false | Prod: don't override manual emergency changes |
| Manual sync required | No | No | Yes | Prod changes need human approval |
| PR required | No | Yes | Yes | All staging/prod changes are reviewed |

---

## Common ArgoCD Commands

```bash
# List all applications
argocd app list

# See full sync status and health
argocd app get myapp-dev

# Manually trigger a sync
argocd app sync myapp-staging

# Rollback to previous version
argocd app rollback myapp-prod

# Force a hard refresh (re-read from Git)
argocd app get myapp-dev --hard-refresh

# Diff: what ArgoCD would change if synced now
argocd app diff myapp-staging

# View application logs
argocd app logs myapp-dev

# Delete an application (does NOT delete cluster resources by default)
argocd app delete myapp-dev
```

---

## Deliverables

| Deliverable | Description |
|---|---|
| `gitops-repo` with full folder structure | Separated per-env Helm values and base chart |
| `myapp-appset.yaml` | Single ApplicationSet managing all three environments |
| `ci.yaml` | GitHub Actions workflow that builds, pushes, and promotes image tags |
| Promotion runbook | Documented process for dev → staging → prod promotion |
| ArgoCD UI | Working dashboard showing all three application sync statuses |

---

## Key Concepts You Now Understand

| Concept | What It Means |
|---|---|
| GitOps | Git is the single source of truth — cluster state is always driven from a repo |
| ApplicationSet | One ArgoCD resource that generates many Applications from a template |
| Sync policy | Rules that control when and how ArgoCD applies changes to the cluster |
| Self-heal | ArgoCD automatically corrects any manual changes made to the cluster |
| Drift | When the live cluster state differs from what Git says it should be |
| Promotion | Moving a validated artifact (image tag) from one environment to the next |
| Prune | Deleting cluster resources that have been removed from Git |

---

## What to Explore Next

- Add image update automation with **Argo Image Updater** (eliminates the sed-based tag update)
- Implement **RBAC** in ArgoCD to restrict who can sync prod
- Add **Kyverno** or **OPA Gatekeeper** for policy enforcement on your clusters
- Move toward the **Guru** project: Internal Developer Platform built on top of this GitOps foundation
