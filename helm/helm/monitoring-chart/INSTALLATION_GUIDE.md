# Monitoring Stack - Installation Guide

## 🎯 Overview

Complete monitoring solution created from `kubernetes-AKS/simple-monitoring` and converted to Helm chart format. This chart deploys a full observability stack alongside your PricePointScout application.

## 📦 What Gets Deployed

The chart creates **19 Kubernetes resources**:

| Resource Type | Count | Components |
|--------------|-------|------------|
| Namespace | 1 | `monitoring` |
| Deployments | 3 | Prometheus, Grafana, Kube State Metrics |
| DaemonSet | 1 | Node Exporter (runs on every node) |
| Services | 4 | Prometheus, Grafana, Node Exporter, KSM |
| ConfigMaps | 3 | Prometheus config, Grafana datasources, Grafana dashboards |
| ServiceAccounts | 2 | Prometheus, Kube State Metrics |
| ClusterRoles | 2 | Prometheus RBAC, KSM RBAC |
| ClusterRoleBindings | 2 | Prometheus, Kube State Metrics |
| PersistentVolumeClaim | 1 | Prometheus storage (10Gi) |

## 🚀 Installation Steps

### 1. Verify Prerequisites

```bash
# Check Helm version
helm version

# Check cluster access
kubectl cluster-info

# Check storage class (should show do-block-storage)
kubectl get storageclass
```

### 2. Review Configuration

Edit `values.yaml` if needed:

```bash
cd /media/omar/01DADC72FB780420/Projects/PricePointScout/helm/helm-tutorial/monitoring-chart
cat values.yaml
```

**Key settings to review:**
- `grafana.adminPassword`: Change from default `admin123`
- `prometheus.storage.size`: Default is 10Gi
- `prometheus.targetNamespaces`: Ensure includes your app namespace

### 3. Install the Chart

```bash
# From the monitoring-chart directory
helm install monitoring . -n monitoring --create-namespace

# Or with custom values
helm install monitoring . -n monitoring --create-namespace \
  --set grafana.adminPassword=your-secure-password \
  --set prometheus.storage.size=20Gi
```

### 4. Verify Installation

```bash
# Check all pods are running
kubectl get pods -n monitoring

# Expected output:
# NAME                                  READY   STATUS    RESTARTS   AGE
# prometheus-xxxxx                      1/1     Running   0          2m
# grafana-xxxxx                         1/1     Running   0          2m
# node-exporter-xxxxx                   1/1     Running   0          2m  (per node)
# kube-state-metrics-xxxxx              1/1     Running   0          2m

# Check services
kubectl get svc -n monitoring

# Check PVC status
kubectl get pvc -n monitoring
```

### 5. Access Grafana

```bash
# Get the LoadBalancer external IP
kubectl get svc grafana -n monitoring

# Wait for EXTERNAL-IP to be assigned (may take 1-2 minutes)
# Example output:
# NAME      TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)
# grafana   LoadBalancer   10.245.123.45   165.227.xxx.xxx   3000:xxxxx/TCP
```

**Access Grafana:**
- URL: `http://<EXTERNAL-IP>:3000`
- Username: `admin`
- Password: `admin123` (or your custom password)

### 6. Access Prometheus

**Option 1: Port Forward (recommended for security)**
```bash
kubectl port-forward svc/prometheus-service 9090:9090 -n monitoring
# Access at http://localhost:9090
```

**Option 2: Change service to LoadBalancer**
```bash
# Edit values.yaml and change:
# prometheus:
#   service:
#     type: LoadBalancer

helm upgrade monitoring . -n monitoring
kubectl get svc prometheus-service -n monitoring
```

## 📊 Verify Metrics Collection

### Check Prometheus Targets

1. Access Prometheus UI (via port-forward)
2. Go to **Status → Targets**
3. Verify all targets are **UP**:
   - `prometheus` (1/1)
   - `node-exporter` (shows all cluster nodes)
   - `kube-state-metrics` (1/1)
   - `kubernetes-cadvisor` (all nodes)
   - `pricepointscout` (shows app pods from pps-namespace)
   - `kubernetes-apiservers` (1/1)

### Check Grafana Dashboards

1. Login to Grafana
2. Go to **Dashboards**
3. Open **PricePointScout Application Monitoring**
4. You should see:
   - HTTP request rate graph
   - Response time graph
   - CPU and memory usage
   - Active pods count
   - Error rate

## 🔧 Common Issues and Solutions

### Issue: Prometheus pod CrashLoopBackOff

**Cause**: Storage class not available

**Solution**:
```bash
# Check storage classes
kubectl get storageclass

# Update values.yaml with correct storage class
# For DigitalOcean: do-block-storage
# For Azure AKS: managed-csi
# For AWS EKS: gp2 or gp3

helm upgrade monitoring . -n monitoring
```

### Issue: Grafana shows "No data"

**Cause**: Datasource not connected to Prometheus

**Solution**:
```bash
# Check if prometheus-service is accessible
kubectl get svc prometheus-service -n monitoring

# Verify datasource configuration
kubectl get cm grafana-datasources -n monitoring -o yaml | grep url

# Should show: url: http://prometheus-service:9090

# Restart Grafana pod
kubectl delete pod -l app=grafana -n monitoring
```

### Issue: No metrics from PricePointScout app

**Cause**: App doesn't have `/metrics` endpoint or wrong namespace

**Solution**:
```bash
# Test if app exposes metrics
kubectl port-forward svc/pricepointscout-service 8080:8080 -n pps-namespace
curl http://localhost:8080/metrics

# Check if pps-namespace is in targetNamespaces
helm get values monitoring -n monitoring

# If not, update values.yaml and upgrade:
# prometheus:
#   targetNamespaces:
#     - pps-namespace
#     - monitoring

helm upgrade monitoring . -n monitoring
```

### Issue: Node Exporter not running on all nodes

**Cause**: Node taints blocking DaemonSet

**Solution**:
```bash
# Check DaemonSet status
kubectl describe daemonset node-exporter -n monitoring

# The chart includes tolerations, but verify:
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### Issue: PVC stuck in Pending

**Cause**: Storage class doesn't exist or has no provisioner

**Solution**:
```bash
# Check PVC events
kubectl describe pvc prometheus-pvc -n monitoring

# Verify storage class exists and is default
kubectl get storageclass

# If using wrong storage class, update values.yaml:
# prometheus:
#   storage:
#     storageClass: "correct-storage-class-name"

# Delete PVC and recreate
kubectl delete pvc prometheus-pvc -n monitoring
helm upgrade monitoring . -n monitoring
```

## 🔄 Upgrade the Chart

After modifying `values.yaml`:

```bash
helm upgrade monitoring . -n monitoring

# Or with specific values
helm upgrade monitoring . -n monitoring \
  --set prometheus.storage.size=50Gi \
  --set grafana.replicas=2
```

## 🗑️ Uninstall

### Remove the monitoring stack:

```bash
# Uninstall the chart
helm uninstall monitoring -n monitoring

# Delete the namespace (this deletes all resources including PVC)
kubectl delete namespace monitoring
```

### Keep data and reinstall later:

```bash
# Uninstall but keep the namespace
helm uninstall monitoring -n monitoring

# PVC will remain (Prometheus data is preserved)
kubectl get pvc -n monitoring

# Reinstall later
helm install monitoring . -n monitoring
# Will reattach to existing PVC
```

## 📈 Next Steps

1. **Customize Dashboards**: Login to Grafana and create custom dashboards for your specific metrics
2. **Set up Alerts**: Configure Prometheus alerting rules
3. **Add More Targets**: Update `prometheus.targetNamespaces` to monitor additional namespaces
4. **Secure Access**: Use Ingress with TLS for Grafana instead of LoadBalancer
5. **Integrate with Alertmanager**: Add alerting capabilities to Prometheus

## 📚 Useful Commands

```bash
# View all resources in monitoring namespace
kubectl get all -n monitoring

# Check Prometheus storage usage
kubectl exec -it deployment/prometheus -n monitoring -- df -h /prometheus

# View Prometheus logs
kubectl logs -f deployment/prometheus -n monitoring

# View Grafana logs
kubectl logs -f deployment/grafana -n monitoring

# Get Helm release info
helm list -n monitoring
helm get values monitoring -n monitoring
helm get manifest monitoring -n monitoring

# Test prometheus scrape configs
kubectl exec -it deployment/prometheus -n monitoring -- wget -O- http://localhost:9090/api/v1/targets

# Restart all monitoring components
kubectl rollout restart deployment -n monitoring
kubectl rollout restart daemonset node-exporter -n monitoring
```

## 🎓 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Monitoring Namespace                     │
│                                                              │
│  ┌──────────────┐      ┌──────────────┐                    │
│  │  Prometheus  │◄─────│  Prometheus  │                    │
│  │  Deployment  │      │  ConfigMap   │                    │
│  └──────┬───────┘      └──────────────┘                    │
│         │                                                    │
│         │ Scrapes metrics from:                             │
│         │                                                    │
│         ├──► Node Exporter (DaemonSet on every node)       │
│         ├──► Kube State Metrics (Deployment)               │
│         ├──► PricePointScout Pods (pps-namespace)          │
│         ├──► Kubernetes API Server                         │
│         └──► Container cAdvisor (kubelet)                  │
│                                                              │
│  ┌──────────────┐                                          │
│  │   Grafana    │                                          │
│  │  Deployment  │─────► Prometheus Service (datasource)   │
│  │              │◄───── Dashboards ConfigMap              │
│  └──────────────┘                                          │
│                                                              │
│  LoadBalancer Service ──► Internet (Port 3000)             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   PPS-Namespace (App)                        │
│                                                              │
│  PricePointScout Pods expose /metrics endpoint              │
│  Discovered by Prometheus via kubernetes_sd_configs         │
└─────────────────────────────────────────────────────────────┘
```

## 🔐 Security Notes

⚠️ **Default configuration is for development/testing**

For production:
1. Change Grafana admin password
2. Use ClusterIP for services and expose via Ingress with TLS
3. Enable authentication for Prometheus
4. Restrict RBAC permissions if monitoring specific namespaces only
5. Use Kubernetes Secrets for credentials instead of values.yaml
6. Enable network policies to restrict access
7. Configure backup for Prometheus PVC data

## 📞 Support

For issues:
1. Check this guide's troubleshooting section
2. View pod logs: `kubectl logs -n monitoring <pod-name>`
3. Check events: `kubectl get events -n monitoring --sort-by='.lastTimestamp'`
4. Validate chart: `helm lint .`
5. Test template rendering: `helm template monitoring . --debug`

---

**Chart Version:** 1.0.0  
**Kubernetes Version:** Tested on v1.28+  
**Helm Version:** Requires Helm 3.x  
**Cloud Provider:** Optimized for DigitalOcean (also works on Azure AKS, AWS EKS)
