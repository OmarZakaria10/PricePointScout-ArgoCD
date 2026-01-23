# Monitoring Stack Helm Chart

A comprehensive monitoring solution for Kubernetes clusters using Prometheus, Grafana, Node Exporter, and Kube State Metrics.

## 📦 Components

- **Prometheus**: Metrics collection and storage with 7-day retention
- **Grafana**: Visualization dashboards with pre-configured PricePointScout dashboard
- **Node Exporter**: System-level metrics from every cluster node
- **Kube State Metrics**: Kubernetes object metrics (pods, deployments, services, etc.)

## 🚀 Quick Start

### Prerequisites

- Kubernetes cluster (tested on DigitalOcean Kubernetes)
- Helm 3.x installed
- `kubectl` configured to access your cluster

### Installation

1. **Install the monitoring stack:**

```bash
cd /media/omar/01DADC72FB780420/Projects/PricePointScout/helm/helm-tutorial/monitoring-chart

helm install monitoring . -n monitoring --create-namespace
```

2. **Verify the installation:**

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Expected pods:
- `prometheus-*` (1 replica)
- `grafana-*` (1 replica)
- `node-exporter-*` (1 per node - DaemonSet)
- `kube-state-metrics-*` (1 replica)

### Access Grafana

Get the Grafana LoadBalancer IP:

```bash
kubectl get svc grafana -n monitoring
```

Access Grafana at: `http://<EXTERNAL-IP>:3000`

**Default Credentials:**
- Username: `admin`
- Password: `admin123` (⚠️ **Change this in production!**)

### Access Prometheus

Forward Prometheus port to access the UI:

```bash
kubectl port-forward svc/prometheus-service 9090:9090 -n monitoring
```

Access Prometheus at: `http://localhost:9090`

## 🎯 Monitoring Targets

The monitoring stack automatically scrapes metrics from:

1. **Prometheus itself** - Self-monitoring
2. **Node Exporter** - CPU, memory, disk, network metrics from all nodes
3. **Kube State Metrics** - Pod status, deployment status, resource quotas
4. **Kubernetes cAdvisor** - Container metrics via kubelet
5. **PricePointScout Application** - Custom app metrics from `/metrics` endpoint
6. **Kubernetes API Server** - API server health and performance

## ⚙️ Configuration

### Key Values

Edit `values.yaml` to customize your deployment:

```yaml
# Namespace for all monitoring components
namespace: monitoring

# Prometheus configuration
prometheus:
  enabled: true
  replicas: 1
  scrapeInterval: 15s
  retention: 7d
  storage:
    enabled: true
    size: 10Gi
    storageClass: "do-block-storage"
  targetNamespaces:
    - pps-namespace  # Scrape PricePointScout app
    - monitoring

# Grafana configuration
grafana:
  enabled: true
  adminUser: admin
  adminPassword: admin123  # ⚠️ CHANGE THIS!
  service:
    type: LoadBalancer  # or ClusterIP for internal only
```

### Common Customizations

**Change Grafana password:**
```yaml
grafana:
  adminPassword: "your-secure-password"
```

**Increase Prometheus storage:**
```yaml
prometheus:
  storage:
    size: 50Gi
```

**Change Grafana service to ClusterIP:**
```yaml
grafana:
  service:
    type: ClusterIP
```

**Monitor additional namespaces:**
```yaml
prometheus:
  targetNamespaces:
    - pps-namespace
    - monitoring
    - your-namespace
```

## 📊 Pre-configured Dashboards

The chart includes a PricePointScout application dashboard with:

- HTTP request rate
- HTTP response time (p95)
- Pod CPU usage
- Pod memory usage
- Active pods count
- Error rate

Access it in Grafana under: **Dashboards → PricePointScout Application Monitoring**

## 🔧 Troubleshooting

### Prometheus not scraping targets

1. Check Prometheus targets:
```bash
kubectl port-forward svc/prometheus-service 9090:9090 -n monitoring
# Visit http://localhost:9090/targets
```

2. Verify RBAC permissions:
```bash
kubectl get clusterrole prometheus
kubectl get clusterrolebinding prometheus
```

### Grafana can't connect to Prometheus

Check the datasource configuration:
```bash
kubectl get cm grafana-datasources -n monitoring -o yaml
```

Ensure the URL is: `http://prometheus-service:9090`

### Node Exporter not running on all nodes

Check DaemonSet status:
```bash
kubectl get daemonset node-exporter -n monitoring
kubectl describe daemonset node-exporter -n monitoring
```

### Storage issues

Check PVC status:
```bash
kubectl get pvc -n monitoring
kubectl describe pvc prometheus-pvc -n monitoring
```

Ensure `storageClass: "do-block-storage"` matches your cluster's storage class:
```bash
kubectl get storageclass
```

## 🔄 Upgrade

To upgrade the monitoring stack:

```bash
helm upgrade monitoring . -n monitoring
```

## 🗑️ Uninstall

To remove the monitoring stack:

```bash
helm uninstall monitoring -n monitoring
```

**Note:** This will NOT delete the PVC by default. To delete it manually:
```bash
kubectl delete pvc prometheus-pvc -n monitoring
```

## 🔐 Security Considerations

### Production Checklist

- [ ] Change Grafana admin password
- [ ] Use Kubernetes Secrets for sensitive values
- [ ] Enable TLS for Grafana (use Ingress)
- [ ] Restrict Grafana service to ClusterIP (use Ingress for external access)
- [ ] Review RBAC permissions (currently uses ClusterRole for cross-namespace monitoring)
- [ ] Enable authentication for Prometheus (use Ingress with auth)
- [ ] Configure network policies to restrict access

### Using Secrets for Credentials

Instead of plain text passwords in `values.yaml`, use Kubernetes Secrets:

```bash
kubectl create secret generic grafana-credentials \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=your-secure-password \
  -n monitoring
```

Then modify the Grafana deployment to use the secret (requires template customization).

## 📈 Metrics Examples

### Query Prometheus

Example PromQL queries for PricePointScout:

```promql
# Request rate
rate(http_requests_total{job="pricepointscout"}[5m])

# P95 response time
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{job="pricepointscout"}[5m]))

# Pod CPU usage
rate(container_cpu_usage_seconds_total{namespace="pps-namespace",pod=~"pricepointscout-.*"}[5m])

# Pod memory usage
container_memory_usage_bytes{namespace="pps-namespace",pod=~"pricepointscout-.*"}

# Active pods
count(kube_pod_status_phase{namespace="pps-namespace",phase="Running"})
```

## 🤝 Integration with PricePointScout

This chart is designed to work alongside the `pricepointscout-chart`. Install both:

```bash
# Install the application
helm install pps ../pricepointscout-chart -n pps-namespace --create-namespace

# Install monitoring
helm install monitoring . -n monitoring --create-namespace
```

The Prometheus configuration automatically discovers and scrapes metrics from the `pps-namespace`.

## 📝 Chart Structure

```
monitoring-chart/
├── Chart.yaml                                    # Chart metadata
├── values.yaml                                   # Default configuration
├── README.md                                     # This file
└── templates/
    ├── namespace.yaml                            # Monitoring namespace
    ├── prometheus-configmap.yaml                 # Prometheus scrape configs
    ├── prometheus-deployment.yaml                # Prometheus deployment
    ├── prometheus-service.yaml                   # Prometheus service
    ├── prometheus-pvc.yaml                       # Prometheus storage
    ├── prometheus-rbac.yaml                      # Prometheus RBAC
    ├── grafana-datasource-configmap.yaml         # Grafana datasources
    ├── grafana-dashboards-configmap.yaml         # Pre-configured dashboards
    ├── grafana-deployment.yaml                   # Grafana deployment
    ├── grafana-service.yaml                      # Grafana service
    ├── node-exporter-daemonset.yaml              # Node Exporter
    ├── node-exporter-service.yaml                # Node Exporter service
    ├── kube-state-metrics-deployment.yaml        # Kube State Metrics
    ├── kube-state-metrics-service.yaml           # KSM service
    └── kube-state-metrics-rbac.yaml              # KSM RBAC
```

## 📚 Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Node Exporter](https://github.com/prometheus/node_exporter)
- [Kube State Metrics](https://github.com/kubernetes/kube-state-metrics)

## 🐛 Support

For issues or questions:
1. Check the troubleshooting section above
2. Review Prometheus targets: `http://localhost:9090/targets`
3. Check pod logs: `kubectl logs -n monitoring <pod-name>`
4. Verify RBAC permissions
5. Ensure storage class is correct for your cluster

## 📄 License

This monitoring chart is part of the PricePointScout project.
