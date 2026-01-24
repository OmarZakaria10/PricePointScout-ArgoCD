# Prerequisites Installation

## 1. Install cert-manager

```bash
# Install cert-manager CRDs and controller
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Verify installation
kubectl get pods -n cert-manager
```

## 2. Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verify installation
kubectl get pods -n argocd

# Optional: Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 3. After Installation

Once both are installed and running, enable them in your values.yaml:

```yaml
argocd:
  enabled: true

ingress:
  tls:
    enabled: true
```

Then install the chart:

```bash
helm install pricepointscout ./helm/pricepointscout-chart/ -n pps-namespace --create-namespace
```
