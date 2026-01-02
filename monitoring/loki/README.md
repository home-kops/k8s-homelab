# Loki

Log aggregation system storing and indexing container logs.

## Configuration

### Deployment Mode

SingleBinary mode with all components in a single pod.

### Storage

10Gi NFS PVC with filesystem backend:
```yaml
storage:
  type: filesystem
persistence:
  size: 10Gi
  storageClass: nfs-sc
```

### Schema

TSDB schema:
```yaml
schemaConfig:
  configs:
    - from: "2024-01-01"
      store: tsdb
      schema: v13
```

### Pattern Ingester

```yaml
pattern_ingester:
  enabled: true
```

## Resources

- **CPU**: 200m requests, 500m limits
- **Memory**: 512Mi requests, 1Gi limits
- **Storage**: 10Gi PVC
- **Node Selector**: `size: m`

## Access

`http://loki.monitoring:3100` with `X-Scope-OrgID: homelab` header.

## References

- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Helm Chart](https://github.com/grafana/loki/tree/main/production/helm/loki)

## Storage and Retention

Logs take up space. Configure retention:

```yaml
limits_config:
  retention_period: 168h  # 7 days
```

After retention period, logs are deleted automatically.

**Storage sizing**:
- Small cluster: ~5-10GB/day
- Medium cluster: ~20-50GB/day
- Depends on: log volume, retention period

## Monitoring Loki

Check Loki status:
```bash
kubectl get pods -n monitoring -l app=loki
kubectl logs -n monitoring -l app=loki
```

Metrics exposed by Loki:
```promql
# Ingestion rate
rate(loki_distributor_bytes_received_total[5m])

# Query performance
histogram_quantile(0.99, rate(loki_request_duration_seconds_bucket[5m]))
```

## Common Log Patterns

### Finding Errors

```logql
{namespace="media"} |= "error" or |= "Error" or |= "ERROR"
```

### HTTP Access Logs

```logql
{app="traefik"} | json | status_code >= 500
```

### Application Startup

```logql
{pod=~"jellyfin.*"} |= "started" or |= "listening"
```

### Failed Authentications

```logql
{namespace="vault"} |= "authentication failed"
```

## Grafana Dashboard Integration

Add Loki queries to Grafana dashboards:

```yaml
panels:
  - title: "Error Logs"
    type: logs
    targets:
      - expr: '{namespace="$namespace"} |= "error"'
        refId: A
```

Or use the built-in log panel for easy browsing.

## Troubleshooting

**No logs appearing**:
- Check log collector (Alloy) is running
- Verify Loki service is accessible
- Check collector configuration

**Query timeout**:
- Reduce time range
- Add more specific label filters
- Increase query limits

**High resource usage**:
- Reduce retention period
- Limit log ingestion rate
- Add more resources to Loki

**Missing logs**:
- Check pod logs are written to STDOUT/STDERR
- Verify namespace label selectors
- Check log collector is on all nodes (DaemonSet)

## Best Practices

- **Log to STDOUT/STDERR**: Kubernetes captures these automatically
- **Structured logging**: Use JSON for easier parsing
- **Include context**: Namespace, app name, pod name in labels
- **Avoid high cardinality labels**: No unique IDs in labels
- **Set retention**: Balance storage costs vs retention needs
- **Monitor ingestion**: Watch for log volume spikes
- **Use log levels**: INFO, WARN, ERROR for filtering

## Structured Logging Example

Instead of:
```
User login failed
```

Use JSON:
```json
{
  "level": "error",
  "event": "login_failed",
  "user": "alice",
  "ip": "192.168.1.100",
  "timestamp": "2024-01-02T10:30:00Z"
}
```

Query:
```logql
{app="myapp"} | json | event="login_failed"
```

## Log Levels

Configure applications to log appropriately:
- **DEBUG**: Development only, very verbose
- **INFO**: General information, normal operations
- **WARN**: Something unusual but not an error
- **ERROR**: An error occurred
- **FATAL**: Critical failure, application stopping

Filter in Loki:
```logql
{namespace="media"} | json | level="error"
```

## Resources

- LogQL guide: https://grafana.com/docs/loki/latest/logql/
- Best practices: https://grafana.com/docs/loki/latest/best-practices/
- Loki docs: https://grafana.com/docs/loki/latest/
