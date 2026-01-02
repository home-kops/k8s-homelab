# Loki

Log aggregation system designed to be cost-effective and easy to operate.

## What is Loki?

Loki collects logs from all containers in your cluster and makes them searchable. Think of it as "Prometheus for logs" - it uses similar labels and integrates perfectly with Grafana.

## What it does

- **Log aggregation**: Collects logs from all pods
- **Indexed labels**: Indexes metadata, not log content (cost-effective)
- **LogQL queries**: Query language similar to PromQL
- **Grafana integration**: Seamless log viewing in Grafana
- **Multi-tenancy**: Support for multiple tenants
- **Long-term storage**: Compress and store logs cheaply

## How it works

1. Applications write logs to STDOUT/STDERR
2. Kubernetes captures container logs
3. Log collector (Alloy/Promtail) reads logs from nodes
4. Collector adds labels (namespace, pod name, etc.)
5. Logs are sent to Loki
6. Loki indexes labels (not log content)
7. You query logs using LogQL in Grafana

## Why Loki?

Traditional log systems (ELK Stack) index all log content:
- ❌ Expensive to run
- ❌ High resource requirements
- ❌ Complex to operate

Loki only indexes labels:
- ✅ Lightweight and cheap
- ✅ Low resource usage
- ✅ Simple to operate
- ✅ Perfect for Kubernetes

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
loki:
  storage:
    type: filesystem
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
      
  retention_period: 168h  # 7 days
  
  limits_config:
    retention_period: 168h
    max_query_series: 10000
```

## LogQL Query Language

Query logs similar to PromQL:

### Basic Queries

```logql
# All logs from media namespace
{namespace="media"}

# Logs from specific pod
{pod="jellyfin-abc123"}

# Logs containing "error"
{namespace="media"} |= "error"

# Logs NOT containing "debug"
{namespace="media"} != "debug"

# Case-insensitive search
{namespace="media"} |~ "(?i)error"
```

### Filtering and Parsing

```logql
# Parse JSON logs
{app="myapp"} | json

# Parse and filter by field
{app="myapp"} | json | level="error"

# Extract fields with regex
{app="myapp"} | regexp "user=(?P<user>\\w+)"

# Line format
{app="myapp"} | line_format "{{.timestamp}} {{.message}}"
```

### Aggregations

```logql
# Count log lines per second
rate({namespace="media"}[5m])

# Count errors per pod
sum(rate({namespace="media"} |= "error" [5m])) by (pod)

# Bytes per second
sum(rate({namespace="media"}[5m])) by (namespace)
```

## Log Labels

Loki automatically adds labels:
- `namespace`: Kubernetes namespace
- `pod`: Pod name
- `container`: Container name
- `stream`: stdout or stderr
- `node`: Node name

You can add custom labels, but **avoid high cardinality**:
- ✅ Good: `app`, `environment`, `level`
- ❌ Bad: `user_id`, `request_id`, `timestamp`

## Deployment

Deployed by ArgoCD along with log collector (Alloy).

## Accessing Logs

### Via Grafana (Recommended)

1. Open Grafana
2. Go to Explore
3. Select Loki data source
4. Use Log Browser to build queries
5. Click on logs to expand details

### Direct API

```bash
# Port-forward to Loki
kubectl port-forward -n monitoring svc/loki 3100:3100

# Query logs
curl -G -s "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={namespace="media"}' \
  --data-urlencode 'limit=10'
```

## Log Collectors

Loki needs a log collector to send logs:

### Alloy (Recommended - included in this repo)
- Modern collector
- Low resource usage
- Configuration as code

### Promtail
- Official Loki agent
- Lightweight
- Good for simple deployments

### Fluentd/Fluent Bit
- More features
- Higher resource usage
- Good for complex pipelines

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
