# Grafana

Visualization and analytics platform for creating dashboards from your metrics, logs, and traces.

## What is Grafana?

Grafana is your monitoring dashboard. It takes metrics from Prometheus, logs from Loki, and traces from Tempo, and displays them in beautiful, interactive visualizations.

## What it does

- **Dashboard creation**: Build custom visualizations of your data
- **Multiple data sources**: Query Prometheus, Loki, Tempo, and more
- **Alerting**: Visual alert rules with notification channels
- **Templating**: Create dynamic, reusable dashboards
- **Explore**: Ad-hoc query and data exploration
- **Sharing**: Share dashboards with your team

## How it works

1. You access Grafana's web interface
2. Grafana connects to configured data sources (Prometheus, Loki, Tempo)
3. You create dashboards with panels (graphs, tables, stats)
4. Each panel runs queries against data sources
5. Grafana renders the results as visualizations
6. Dashboards auto-refresh to show real-time data

## Key Features

- **Unified observability**: Metrics, logs, traces in one interface
- **Correlation**: Jump from metric spike to related logs
- **Annotations**: Mark events on graphs
- **Variables**: Make dashboards dynamic with dropdowns
- **Playlists**: Rotate through dashboards on a TV
- **Alerting**: Define alerts visually

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
persistence:
  enabled: true
  size: 10Gi
  storageClass: nfs-sc

adminPassword: <path:secret/data/monitoring/grafana#admin_password>

datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus-server
    
  - name: Loki
    type: loki
    url: http://loki:3100
    
  - name: Tempo
    type: tempo
    url: http://tempo:3100
```

## Accessing Grafana

Access through your configured ingress URL (e.g., `https://grafana.yourdomain.com`)

Default credentials:
- Username: `admin`
- Password: Stored in Vault or configured in `values.yaml`

## Creating Dashboards

### 1. Add a Data Source
- Go to Configuration → Data Sources
- Add Prometheus, Loki, Tempo
- Test connection

### 2. Create Dashboard
- Click + → Dashboard
- Add Panel
- Select visualization type (Graph, Gauge, Table, etc.)
- Write query in PromQL (for Prometheus) or LogQL (for Loki)
- Configure panel settings

### 3. Example Panel

**CPU Usage by Pod**:
```promql
sum(rate(container_cpu_usage_seconds_total{namespace="media"}[5m])) by (pod)
```

**Application Logs**:
```logql
{namespace="media"} |= "error"
```

## Pre-built Dashboards

Import community dashboards from Grafana.com:

Popular dashboards:
- Kubernetes Cluster Monitoring (ID: 7249)
- Node Exporter Full (ID: 1860)
- Traefik (ID: 11462)
- Prometheus Stats (ID: 2)

Import: Dashboard → Import → Enter ID

## Dashboard Organization

```
Dashboards/
├── Kubernetes/
│   ├── Cluster Overview
│   ├── Node Details
│   └── Pod Details
├── Applications/
│   ├── Jellyfin
│   ├── Vault
│   └── ArgoCD
├── Networking/
│   └── Traefik
└── Security/
    ├── Falco Alerts
    └── CrowdSec
```

## Variables

Make dashboards dynamic with variables:

```
$namespace: Dropdown to select namespace
$pod: Dropdown to select pod
```

Query:
```promql
container_memory_usage_bytes{namespace="$namespace", pod="$pod"}
```

## Alerting

Create alerts in Grafana:

1. Edit a panel
2. Go to Alert tab
3. Define condition (e.g., "when avg() is above 80")
4. Configure notification channel (Email, Slack, etc.)
5. Save alert

## Explore View

Ad-hoc querying without creating dashboards:
- Switch to Explore view
- Select data source
- Write queries
- View results
- Correlate metrics → logs → traces

## Deployment

Deployed by ArgoCD. Grafana starts with pre-configured data sources.

## Dashboard Storage

Dashboards can be:
- Stored in Grafana's database (default)
- Provisioned from ConfigMaps (GitOps approach)
- Imported from JSON files

GitOps approach (recommended):
```yaml
# dashboards/my-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  my-dashboard.json: |
    {...dashboard JSON...}
```

## Integrations

- **Prometheus**: Metrics and alerting
- **Loki**: Log aggregation and search
- **Tempo**: Distributed tracing
- **AlertManager**: Alert routing
- **Slack/Email**: Notification channels

## Users and Permissions

Create teams and users:
- Admin: Full access
- Editor: Create/edit dashboards
- Viewer: View dashboards only

Configure organizations for multi-tenancy.

## Plugins

Extend Grafana with plugins:
- Visualization plugins (e.g., Worldmap, Pie Chart)
- Data source plugins (e.g., MongoDB, InfluxDB)
- App plugins (e.g., Kubernetes app)

Install plugins in `values.yaml`:
```yaml
plugins:
  - grafana-piechart-panel
  - grafana-worldmap-panel
```

## Annotations

Mark events on dashboards:
- Deployments
- Incidents
- Configuration changes

Queries from Prometheus or Loki can create annotations automatically.

## Monitoring Grafana

Check Grafana status:
```bash
kubectl get pods -n monitoring -l app=grafana
kubectl logs -n monitoring -l app=grafana
```

## Troubleshooting

**Can't login**: Check credentials, verify pod is running

**Data source failing**: Verify data source URL, check network connectivity

**Dashboards not loading**: Check browser console, verify data source queries

**Slow queries**: Optimize PromQL, reduce time range, add query caching

**High memory usage**: Limit dashboard complexity, reduce refresh rates

## Best Practices

- **Use variables**: Make dashboards reusable
- **Folder organization**: Group related dashboards
- **Dashboard links**: Link related dashboards together
- **Provisioning**: Store dashboards as code in Git
- **Alert sparingly**: Too many alerts cause fatigue
- **Standardize**: Use consistent naming and layouts

## Example Dashboard

```json
{
  "dashboard": {
    "title": "Jellyfin Monitoring",
    "panels": [
      {
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total{pod=~\"jellyfin.*\"}[5m])"
          }
        ]
      },
      {
        "title": "Active Streams",
        "type": "stat",
        "targets": [
          {
            "expr": "jellyfin_active_streams"
          }
        ]
      }
    ]
  }
}
```

## Resources

- Dashboard gallery: https://grafana.com/grafana/dashboards/
- Documentation: https://grafana.com/docs/
- PromQL cheat sheet: https://promlabs.com/promql-cheat-sheet/
- LogQL guide: https://grafana.com/docs/loki/latest/logql/
