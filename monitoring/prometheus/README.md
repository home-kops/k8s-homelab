# Prometheus

Monitoring system and time-series database for collecting and storing metrics.

## What is Prometheus?

Prometheus is the metrics engine of your observability stack. It collects performance data from all your applications and Kubernetes components, storing it for querying and alerting.

## What it does

- **Metrics collection**: Scrapes metrics from applications and services
- **Time-series storage**: Stores metrics with timestamps
- **Query language**: PromQL for powerful metric queries
- **Alerting**: Defines rules to trigger alerts on conditions
- **Service discovery**: Automatically finds services to monitor
- **Multi-dimensional data**: Labels for flexible querying

## How it works

1. Prometheus discovers targets to monitor (services, pods, nodes)
2. It scrapes HTTP endpoints on those targets (usually `/metrics`)
3. Metrics are stored in its time-series database
4. You query metrics using PromQL (Prometheus Query Language)
5. Grafana visualizes the metrics
6. AlertManager sends notifications when thresholds are breached

## What Prometheus Monitors

In your cluster:
- **Kubernetes components**: API server, scheduler, controller-manager
- **Nodes**: CPU, memory, disk, network usage
- **Pods**: Container resource usage
- **Applications**: Custom application metrics
- **Exporters**: Specialized metrics (node-exporter, kube-state-metrics)

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
server:
  persistentVolume:
    enabled: true
    size: 8Gi
    storageClass: nfs-sc
    
  retention: "15d"  # Keep metrics for 15 days
  
  resources:
    requests:
      cpu: 250m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi
```

### Components Deployed

- **Prometheus Server**: Main component that scrapes and stores metrics
- **kube-state-metrics**: Exposes Kubernetes object state as metrics
- **node-exporter**: Exposes node hardware/OS metrics
- **AlertManager** (optional): Handles alert routing and notifications

## Metrics Format

Prometheus metrics format:
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234
http_requests_total{method="POST",status="201"} 567
```

## PromQL Queries

Query language for metrics:

```promql
# CPU usage by pod
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)

# Memory usage
container_memory_usage_bytes{namespace="media"}

# HTTP request rate
rate(http_requests_total[5m])

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

## Common Metrics

**Kubernetes**:
- `kube_pod_status_phase`: Pod state (Running, Pending, Failed)
- `kube_deployment_status_replicas`: Deployment replica counts
- `container_memory_usage_bytes`: Container memory usage
- `container_cpu_usage_seconds_total`: Container CPU usage

**Node**:
- `node_cpu_seconds_total`: Node CPU time
- `node_memory_MemAvailable_bytes`: Available memory
- `node_filesystem_avail_bytes`: Disk space available
- `node_network_receive_bytes_total`: Network received

## Deployment

Deployed by ArgoCD. Prometheus starts collecting metrics immediately.

## Accessing Prometheus

Access through ingress or port-forward:
```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:80
```

Then open: `http://localhost:9090`

### Prometheus UI

The web UI allows:
- Execute PromQL queries
- Browse available metrics
- View alert rules
- Check targets (scraped endpoints)
- Graph metrics

## Integrations

- **Grafana**: Visualizes Prometheus metrics in dashboards
- **AlertManager**: Sends alerts based on Prometheus rules
- **ServiceMonitors**: Custom resources for service discovery
- **Remote Write**: Send metrics to external systems

## Retention and Storage

Metrics are kept based on retention settings:
- Default: 15 days
- Configure in `values.yaml`: `server.retention`

Storage grows with:
- Number of metrics
- Scrape frequency
- Retention period
- Cardinality (unique label combinations)

Monitor storage usage:
```bash
kubectl exec -n monitoring prometheus-server-0 -- df -h /data
```

## Creating Alerts

Define alert rules:

```yaml
groups:
  - name: example
    rules:
      - alert: HighPodMemory
        expr: container_memory_usage_bytes > 1000000000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} using high memory"
```

## ServiceMonitor CRD

Define custom scrape targets:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
```

## Monitoring Prometheus

Watch Prometheus itself:
```bash
kubectl get pods -n monitoring -l app=prometheus
kubectl logs -n monitoring prometheus-server-0
```

Check targets in Prometheus UI: Status → Targets

## Troubleshooting

**High memory usage**: Reduce retention period, decrease scrape frequency, reduce cardinality

**Missing metrics**: Check target is being scraped (Status → Targets), verify `/metrics` endpoint

**Slow queries**: Reduce query time range, optimize PromQL expressions, add more resources

**Storage full**: Increase PV size or decrease retention period

## Best Practices

- **Label cardinality**: Avoid high-cardinality labels (e.g., user IDs)
- **Scrape intervals**: 30-60 seconds for most applications
- **Retention**: Keep 15-30 days, use remote storage for longer
- **Resource limits**: Monitor Prometheus resource usage
- **Backup**: Regularly backup Prometheus data

## Federation

For multi-cluster setups, use Prometheus federation to aggregate metrics:

```yaml
- job_name: 'federate'
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
      - '{job="kubernetes-pods"}'
  static_configs:
    - targets:
      - 'prometheus-1:9090'
      - 'prometheus-2:9090'
```

## Resources

- PromQL guide: https://prometheus.io/docs/prometheus/latest/querying/basics/
- Best practices: https://prometheus.io/docs/practices/naming/
- Exporters: https://prometheus.io/docs/instrumenting/exporters/
