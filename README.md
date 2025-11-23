# k8s-homelab

A GitOps-managed Kubernetes homelab deployment repository using ArgoCD for continuous delivery.

## Purpose

This repository contains Kubernetes manifests and Helm charts for deploying a complete homelab infrastructure. All services are managed via GitOps principles using ArgoCD, which continuously monitors this repository and synchronizes the desired state to the cluster.

## Repository Structure

Each top-level directory represents a **Kubernetes namespace** containing related services:

- **`devops-tools/`** - CI/CD and automation infrastructure (ArgoCD, Renovate, GitHub Arc Runners)
- **`media/`** - Media and eBook management (Calibre Web Automated, Jellyfin)
- **`misc/`** - Miscellaneous applications (Homepage, LocalAI, OrcaSlicer)
- **`monitoring/`** - Observability stack (Prometheus, Grafana, Loki, Tempo, Alloy, CrowdSec)
- **`network/`** - Networking infrastructure (Traefik, Cert-Manager, MetalLB, Replicator)
- **`sec/`** - Security tooling (Falco, KubeBench)
- **`storage/`** - Storage solutions (NFS provisioner)
- **`vault/`** - Secrets management (HashiCorp Vault with automated unsealer)

## Bootstrap Process

Critical namespaces include a `tooling/` folder with a `bootstrap` script. These scripts deploy the minimum required services before ArgoCD can take over management:

```bash
# Example: Bootstrap network namespace
./network/tooling/bootstrap

# Bootstrap storage
./storage/tooling/bootstrap

# Bootstrap vault
./vault/tooling/bootstrap
```

**Bootstrap order:**
1. Storage (NFS provisioner)
2. Network infrastructure (MetalLB, Traefik, Cert-Manager)
3. Vault (secrets management)
4. DevOps tools (ArgoCD)

After bootstrapping, ArgoCD automatically manages all subsequent deployments and updates.

## Technologies

- **GitOps:** ArgoCD for continuous deployment
- **Package Management:** Helm charts with Kustomize overlays
- **Ingress:** Traefik with automatic TLS via Cert-Manager
- **Observability:** Full stack with Prometheus, Loki, Grafana, and Tempo
- **Security:** Vault for secrets, Falco for runtime security, CrowdSec for threat detection
- **Automation:** Renovate for dependency updates

## Getting Started

1. Ensure your Kubernetes cluster is running
2. Run bootstrap scripts in order for critical namespaces
3. Deploy ArgoCD: `./devops-tools/tooling/bootstrap`
4. Configure ArgoCD to monitor this repository
5. ArgoCD will handle all subsequent deployments

## Contributing

All changes should be made via pull requests. ArgoCD will automatically sync approved changes to the cluster.
