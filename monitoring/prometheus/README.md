# Prometheus

Metrics collection and time-series database for cluster and application monitoring.

## Configuration

### Storage

8Gi NFS-backed PVC:
```yaml
server:
  persistentVolume:
    size: 8Gi
    storageClass: nfs-sc
```

### Remote Write Receiver

Accepts metrics from external sources:
```yaml
server:
  remoteWrite:
    receiver:
      enabled: true
```

### Exporters

**node-exporter**: DaemonSet collecting host metrics from all nodes.

**kube-state-metrics**: Kubernetes object metrics.

## Resources

- **CPU**: 250m requests, 1000m limits
- **Memory**: 1Gi requests, 2Gi limits
- **Storage**: 8Gi PVC
- **Node Selector**: `size: m`

## Access

`http://prometheus-server.monitoring:80`

## Dependencies

- node-exporter
- kube-state-metrics

## References

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus)

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
