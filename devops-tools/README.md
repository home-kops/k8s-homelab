# DevOps Tools

This module contains CI/CD and automation infrastructure that helps you build, test, and deploy applications automatically.

## What's included?

### [ArgoCD](./argocd/README.md)
The GitOps continuous delivery tool that manages all deployments in your cluster. ArgoCD monitors this Git repository and automatically applies changes to your Kubernetes cluster.

### [Renovate](./renovate/README.md)
Automated dependency update tool that keeps your container images and Helm charts up to date.

### [GitHub Arc Runners](./arcrunners/README.md)
Self-hosted GitHub Actions runners that run directly in your Kubernetes cluster. To be used by private repositories.

## Deployment

This module is typically bootstrapped using:

```bash
./devops-tools/tooling/bootstrap
```

This script should be run **after** storage, network, and vault are deployed, as ArgoCD is the final piece that manages everything else.
