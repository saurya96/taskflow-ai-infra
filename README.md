# TaskFlow AI Infrastructure

This repository contains Kubernetes, Kustomize, and Argo CD configuration for the TaskFlow AI application.

## Prerequisites

- k3s or Kubernetes 1.27+
- `kubectl`
- An ingress-nginx controller
- A default StorageClass for MongoDB and Redis PVCs
- Container registry access to GHCR images
- A Kubernetes Secret named `taskflow-secrets`

## Repository layout

```text
kubernetes/base/                 Shared resources
kubernetes/overlays/staging/     Staging overlay
kubernetes/overlays/production/  Production overlay
argocd/applications/            Argo CD Applications
```

## Create credentials securely

Do not apply `secrets.example.yaml` as production credentials. Create the real Secret after first creating the namespace:

```bash
kubectl create namespace taskflow-ai
kubectl -n taskflow-ai create secret generic taskflow-secrets \
  --from-literal=MONGO_USERNAME='replace-me' \
  --from-literal=MONGO_PASSWORD='replace-me' \
  --from-literal=MONGO_URI='mongodb://replace-me:replace-me@mongodb:27017/taskflow_ai?authSource=admin' \
  --from-literal=REDIS_PASSWORD='replace-me' \
  --from-literal=JWT_SECRET='replace-me'
```

External Secrets or Sealed Secrets is recommended instead of storing the Secret in Git.

## Deploy staging

```bash
kubectl apply -k kubernetes/overlays/staging
kubectl -n taskflow-ai-staging get all,pvc,ingress
```

## Deploy production

```bash
kubectl apply -k kubernetes/overlays/production
kubectl -n taskflow-ai get all,pvc,ingress
```

Expected StatefulSets use persistent volume claims for MongoDB and Redis. Frontend and backend use readiness/liveness probes. The worker deployment supports replica scaling:

```bash
kubectl -n taskflow-ai scale deployment worker --replicas=5
```

## Install Argo CD on k3s

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get pods
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080` and use the `admin` account with the retrieved password.

## Configure the application

The production and staging Argo CD applications already use `https://github.com/saurya96/taskflow-ai-infra.git`, and the base deployments use `ghcr.io/saurya96/taskflow-ai/*`. Create the repository in Argo CD or through the dashboard, then apply:

```bash
kubectl apply -f argocd/applications/taskflow-ai.yaml
kubectl -n argocd get applications
kubectl -n argocd describe application taskflow-ai-production
```

The Application enables prune, self-heal, and automated sync for `kubernetes/overlays/production`.

## Argo CD screenshot

Capture the Argo CD application list and `taskflow-ai-production` tree only after the cluster reports a healthy/synced state. Do not add a screenshot unless it was captured from a real deployment.

## GitOps image updates

The application repository CI workflow may push SHA tags to GHCR and update `kubernetes/base/deployments.yaml` through a PAT. Configure:

- `INFRA_REPO_PAT` secret: fine-grained PAT with Read and Write access to `saurya96/taskflow-ai-infra` Contents
- `INFRA_REPO_URL` variable: `https://github.com/saurya96/taskflow-ai-infra.git` (the workflow has this default)

Create a fine-grained PAT in GitHub under Settings → Developer settings → Personal access tokens, limit it to the infrastructure repository, and grant Contents: Read and write. Add it as a repository Actions secret named `INFRA_REPO_PAT`; never paste it into source or chat.

## Validation

```bash
kubectl kustomize kubernetes/overlays/staging
kubectl kustomize kubernetes/overlays/production
kubectl apply --dry-run=client -k kubernetes/overlays/staging
kubectl apply --dry-run=client -k kubernetes/overlays/production
```

A live Argo CD deployment and live URL require cluster credentials and DNS/Ingress configuration, neither of which is available in this workspace.
