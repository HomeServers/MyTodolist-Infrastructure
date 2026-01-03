# MyTodoList Infrastructure

Kubernetes manifests for MyTodoList application deployment.

## Overview

This repository contains all Kubernetes resources for deploying the MyTodoList application using GitOps with ArgoCD.

**Architecture:**
- Frontend: React 18 + nginx
- Backend: Spring Boot + Kotlin
- Database: MySQL 9.2.0
- Ingress: nginx-ingress-controller
- SSL/TLS: cert-manager + Let's Encrypt
- GitOps: ArgoCD

## Directory Structure

```
MyTodoList-infra/
├── argocd/
│   └── applications/
│       └── todolist-app.yaml         # ArgoCD Application definition
├── base/
│   ├── namespace.yaml                # todolist namespace
│   ├── mysql-statefulset.yaml        # MySQL + PVC + Service + Secret
│   ├── backend-deployment.yaml       # Backend Deployment + Service
│   ├── frontend-deployment.yaml      # Frontend Deployment + Service
│   ├── ingress.yaml                  # Ingress routing
│   └── kustomization.yaml            # Kustomize base config
├── overlays/
│   └── production/
│       └── kustomization.yaml        # Production overlay
└── cert-manager/
    └── clusterissuer.yaml            # Let's Encrypt ClusterIssuers
```

## Prerequisites

1. **Kubernetes Cluster** (v1.28+)
2. **kubectl** configured with cluster access
3. **Helm** (v3.12+) for installing controllers
4. **ArgoCD** for GitOps
5. **cert-manager** for SSL/TLS
6. **nginx-ingress-controller**

## Installation

### 1. Install Required Controllers

```bash
# Install nginx-ingress-controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.1/cert-manager.yaml

# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Configure cert-manager

**IMPORTANT:** Update email in `cert-manager/clusterissuer.yaml` before applying!

```bash
# Edit the file and replace 'your-email@example.com'
vi cert-manager/clusterissuer.yaml

# Apply ClusterIssuers
kubectl apply -f cert-manager/clusterissuer.yaml
```

### 3. Update ArgoCD Application

Edit `argocd/applications/todolist-app.yaml` and update:
- `repoURL`: Your GitHub repository URL
- `targetRevision`: Your branch name (usually `main`)

### 4. Deploy with ArgoCD

```bash
# Apply ArgoCD Application
kubectl apply -f argocd/applications/todolist-app.yaml

# Check application status
kubectl get applications -n argocd

# Sync application (if not auto-synced)
argocd app sync todolist
```

## Domains

- **Frontend**: `todo.nuhgnod.site`, `www.todo.nuhgnod.site`
- **Backend API**: `api.todo.nuhgnod.site`

## Secrets

### Required GitHub Secrets

For CI/CD workflows to update this repository:

- `DOCKER_USERNAME`: Docker Hub username
- `DOCKER_PASSWORD`: Docker Hub password
- `INFRA_REPO_TOKEN`: GitHub Personal Access Token with repo write access

### Kubernetes Secrets

MySQL credentials are stored in `mysql-secret` (base/mysql-statefulset.yaml):
- Root password: `password` (⚠️ Change in production!)
- Database: `main`

## GitOps Workflow

1. Developer pushes code to `MyTodoList-fe` or `MyTodoList-be`
2. Tag version (e.g., `v1.0.0`)
3. GitHub Actions builds and pushes Docker image
4. GitHub Actions updates image tag in this repository
5. ArgoCD detects change and syncs to Kubernetes

## Monitoring

```bash
# Check all resources
kubectl get all -n todolist

# Check pods
kubectl get pods -n todolist -w

# Check certificates
kubectl get certificate -n todolist

# View logs
kubectl logs -f deployment/todolist-backend -n todolist
kubectl logs -f deployment/todolist-frontend -n todolist
```

## Troubleshooting

### Pods not starting

```bash
kubectl describe pod <pod-name> -n todolist
kubectl logs <pod-name> -n todolist
```

### Certificate not issued

```bash
kubectl describe certificate -n todolist
kubectl describe certificaterequest -n todolist
kubectl logs -n cert-manager deployment/cert-manager
```

### Ingress issues

```bash
kubectl describe ingress todolist-ingress -n todolist
kubectl logs -n ingress-nginx deployment/nginx-ingress-ingress-nginx-controller
```

## Backup & Restore

### Backup MySQL Data

```bash
kubectl exec -n todolist mysql-0 -- mysqldump -u root -ppassword main > backup.sql
```

### Restore MySQL Data

```bash
kubectl cp backup.sql todolist/mysql-0:/tmp/
kubectl exec -n todolist mysql-0 -- mysql -u root -ppassword main < /tmp/backup.sql
```

## Updating Application

### Update Image Version

Images are automatically updated by CI/CD pipelines. To manually update:

```bash
cd base
# Edit kustomization.yaml and change newTag
kubectl apply -k .
```

### Rollback

```bash
# Via ArgoCD
argocd app rollback todolist

# Via kubectl
kubectl rollout undo deployment/todolist-backend -n todolist
kubectl rollout undo deployment/todolist-frontend -n todolist
```

## Security Notes

⚠️ **Important:**
- Change default MySQL password before production use
- Review and update Secret management (consider Sealed Secrets)
- Implement NetworkPolicies for pod-to-pod communication
- Enable Pod Security Standards
- Regular security scans of container images

## Contributing

1. Make changes in feature branch
2. Test changes locally with `kubectl apply -k base`
3. Create Pull Request
4. After merge, ArgoCD will auto-deploy

## License

MIT

## Support

For issues, please create an issue in this repository.
