# Grafana

Visualization and analytics platform for metrics, logs, and traces.

## Configuration

### Authentication

Admin credentials from Vault:
```yaml
adminUser: admin
adminPassword: <path:secret/data/monitoring/grafana#admin_password>
```

### Pre-configured Datasources

**Prometheus** (default):
```yaml
url: http://prometheus-server.monitoring:80
```

**Loki** (with multi-tenancy header):
```yaml
url: http://loki.monitoring:3100
jsonData:
  httpHeaderName1: "X-Scope-OrgID"
secureJsonData:
  httpHeaderValue1: "homelab"
```

**Tempo**:
```yaml
url: http://tempo.monitoring:3100
```

### Imported Dashboards

- Traefik (ID: 17347)
- CrowdSec (ID: 21419)
- Falco (ID: 17319)

## Resources

Default Helm chart allocations.

## Access

`https://grafana.{{ .Values.domain }}`

## Dependencies

- Prometheus
- Loki
- Tempo

## References

- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [Helm Chart](https://github.com/grafana/helm-charts/tree/main/charts/grafana)## Explore View

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
