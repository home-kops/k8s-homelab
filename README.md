# k8s-homelab

A GitOps-managed Kubernetes homelab repository for deploying and managing a complete home infrastructure using ArgoCD.

## Overview

This repository contains all the configuration files needed to run a Kubernetes cluster in your homelab. Think of it as a blueprint for your infrastructure - every application, service, and configuration is defined here as code. When you make changes to these files, ArgoCD automatically applies them to your cluster, keeping everything in sync.

## How does it work?

The repository uses **GitOps** principles:
1. You define what you want (applications, configurations) in this repository
2. ArgoCD monitors this repository for changes
3. ArgoCD automatically deploys and updates everything in your cluster to match what's defined here
4. No manual `kubectl apply` commands needed - just commit and push!

## Repository Structure

The repository is organized into modules, each representing a different namespace of the homelab:

### 📦 [devops-tools/](./devops-tools/README.md)
CI/CD and automation tools that help you build and deploy applications.
- **ArgoCD**: The GitOps engine that manages all deployments
- **Renovate**: Automatically updates dependencies and container images
- **GitHub Arc Runners**: Self-hosted GitHub Actions runners

### 🎬 [media/](./media/README.md)
Media management and streaming applications for your home.
- **Jellyfin**: Media streaming server
- **Calibre Web Automated**: eBook management and conversion
- **CWA Downloader**: Automated eBook downloader companion

### 📊 [monitoring/](./monitoring/README.md)
Observability stack to understand what's happening in your cluster.
- **Prometheus**: Collects metrics from all applications
- **Grafana**: Visualizes metrics in beautiful dashboards
- **Loki**: Aggregates and searches logs
- **Tempo**: Distributed tracing for debugging
- **Alloy**: Telemetry collector
- **CrowdSec**: Collaborative security monitoring

### 🌐 [network/](./network/README.md)
Core networking infrastructure for accessing your applications.
- **Traefik**: Reverse proxy and ingress controller
- **Cert-Manager**: Automatic SSL/TLS certificates
- **MetalLB**: Load balancer for bare metal clusters
- **Replicator**: Syncs secrets/configmaps across namespaces

### 🔒 [sec/](./sec/README.md)
Security tools for protecting your cluster.
- **Falco**: Runtime security monitoring
- **KubeBench**: Kubernetes security best practices scanner

### 💾 [storage/](./storage/README.md)
Storage solutions for persistent data.
- **NFS Provisioner**: Dynamic NFS-backed storage volumes

### 🔐 [vault/](./vault/README.md)
Secrets management for keeping sensitive data secure.
- **HashiCorp Vault**: Centralized secrets management
- **Vault Unsealer**: Automated vault unsealing after restarts

## Getting Started

### Prerequisites
- A running Kubernetes cluster
- `kubectl` configured to access your cluster
- Git installed on your local machine

### Initial Deployment

Services must be deployed in a specific order because they depend on each other:

```bash
# 1. Deploy storage (required by other services for persistent data)
./storage/tooling/bootstrap

# 2. Deploy network infrastructure (required for ingress/load balancing)
./network/tooling/bootstrap

# 3. Deploy secrets management (required for secure credential storage)
./vault/tooling/bootstrap

# 4. Configure the vault and secrets loading
ADMIN_PASSWORD="pass" VAULT_TOKEN="token" ./vault/tooling/configure

# 5. Deploy ArgoCD (takes over managing everything else)
./devops-tools/tooling/bootstrap
```

After running these bootstrap scripts, ArgoCD will automatically deploy and manage all other applications defined in this repository.

### Making Changes

1. Edit the files in this repository (e.g., update a version, change configuration)
2. Commit and push your changes
3. ArgoCD automatically detects the changes and updates your cluster
4. Monitor the sync status in the ArgoCD UI

## Key Technologies

- **Kubernetes**: Container orchestration platform
- **ArgoCD**: GitOps continuous delivery tool
- **Helm**: Package manager for Kubernetes applications
- **Kustomize**: Configuration customization tool
- **Traefik**: Modern HTTP reverse proxy and load balancer
- **Prometheus & Grafana**: Monitoring and observability stack

## Need Help?

Each module has its own README with detailed information:
- Check the module's README.md for an overview
- Check individual component READMEs for specific configuration details
- Component `values.yaml` files contain all customizable settings
- **Security:** Vault for secrets, Falco for runtime security, CrowdSec for threat detection
- **Automation:** Renovate for dependency updates

## Getting Started

1. Ensure your Kubernetes cluster is running
2. Run bootstrap scripts in order for critical namespaces
3. Deploy ArgoCD: `./devops-tools/tooling/bootstrap`
4. Configure ArgoCD to monitor this repository
5. ArgoCD will handle all subsequent deployments
