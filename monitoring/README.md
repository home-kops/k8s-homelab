# Monitoring

This module contains the complete observability stack for understanding what's happening in your Kubernetes cluster.

## What's included?

### [Prometheus](./prometheus/README.md)
A time-series database that collects and stores metrics from all your applications and Kubernetes components.

**What it does**: Scrapes metrics endpoints and stores performance data (CPU, memory, request rates, error rates, etc.).

### [Grafana](./grafana/README.md)
A visualization platform that creates beautiful dashboards from your metrics data.

**What it does**: Displays your Prometheus metrics in graphs, charts, and alerts. Access it through a web browser to see cluster health.

### [Loki](./loki/README.md)
A log aggregation system designed to work seamlessly with Grafana.

**What it does**: Collects logs from all containers in your cluster and makes them searchable in Grafana.

### [Tempo](./tempo/README.md)
A distributed tracing backend for understanding request flows across microservices.

**What it does**: Tracks individual requests as they travel through multiple services, helping debug performance issues.

### [Alloy](./alloy/README.md)
OpenTelemetry collector that gathers telemetry data (metrics, logs, traces) from applications.

**What it does**: Collects and forwards observability data to Prometheus, Loki, and Tempo.

### [CrowdSec](./crowdsec/README.md)
A collaborative security engine that detects and blocks malicious behavior.

**What it does**: Analyzes logs to detect attacks, shares threat intelligence with the community, and can block attackers at the Traefik level.

## How they work together

This is a complete observability stack following industry best practices:

1. **Alloy** collects metrics, logs, and traces from your applications
2. **Prometheus** stores metrics, **Loki** stores logs, **Tempo** stores traces
3. **Grafana** provides a unified interface to query and visualize all three
4. **CrowdSec** analyzes logs for security threats and integrates with Traefik to block attacks

This gives you the "three pillars of observability":
- **Metrics**: What is slow?
- **Logs**: What is the error?
- **Traces**: Where is the problem?

## Deployment

These services are managed by ArgoCD. Each component has a `values.yaml` file with customizable settings.

**Important**: Most monitoring components require persistent storage (provided by NFS) to retain historical data.

## Accessing Grafana

Once deployed, access Grafana through the ingress URL defined in your configuration. Default credentials are typically stored in Kubernetes secrets.
