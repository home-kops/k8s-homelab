# Security (sec)

This module contains security tools that monitor and protect your Kubernetes cluster from threats and misconfigurations.

## What's included?

### [Falco](./falco/README.md)
Runtime security monitoring system that detects unexpected behavior in your cluster.

**What it does**: Monitors system calls and Kubernetes events in real-time, alerting you to suspicious activity like:
- Unexpected file access
- Privilege escalation attempts
- Suspicious network connections
- Container escape attempts

### [KubeBench](./kubebench/README.md)
Security scanner that checks your cluster against CIS Kubernetes Benchmark standards.

**What it does**: Runs automated tests to verify your cluster follows security best practices defined by the Center for Internet Security (CIS).

## How they work together

These tools provide complementary security layers:

- **Falco**: Runtime monitoring - detects active threats and anomalies
- **KubeBench**: Configuration audit - identifies security weaknesses in your setup

Together, they help you maintain a secure Kubernetes environment by catching both configuration mistakes and runtime attacks.

## Deployment

These services are managed by ArgoCD:

- **Falco** runs as a DaemonSet on every node, continuously monitoring
- **KubeBench** typically runs as a Job or CronJob, performing periodic scans

## Reviewing Alerts

- **Falco alerts**: Check your logging system (Loki/Grafana) or Falco's configured output
- **KubeBench results**: Review Job logs after each scan runs

## Important Notes

**Falco** requires privileged access to the host system to monitor kernel events. This is normal and necessary for its operation.

**KubeBench** results should be reviewed but not all findings may apply to homelab environments. Prioritize fixes based on your threat model.
