# DevOps Tools

This module contains CI/CD and automation infrastructure that helps you build, test, and deploy applications automatically.

## What's included?

### [ArgoCD](./argocd/README.md)
The GitOps continuous delivery tool that manages all deployments in your cluster. ArgoCD monitors this Git repository and automatically applies changes to your Kubernetes cluster.

**What it does**: Keeps your cluster in sync with this repository - when you commit changes, ArgoCD deploys them automatically.

### [Renovate](./renovate/README.md)
Automated dependency update tool that keeps your container images and Helm charts up to date.

**What it does**: Scans your manifests, finds outdated versions, and creates pull requests with updates.

### [GitHub Arc Runners](./arcrunners/README.md)
Self-hosted GitHub Actions runners that run directly in your Kubernetes cluster.

**What it does**: Executes your GitHub Actions workflows on your own infrastructure instead of GitHub's cloud runners.

## How they work together

1. **Renovate** detects when a new container image version is available
2. **Renovate** creates a pull request with the update
3. You review and merge the pull request
4. **ArgoCD** detects the change in the repository
5. **ArgoCD** automatically deploys the updated version to your cluster
6. **Arc Runners** can execute any CI/CD pipelines you define in GitHub Actions

## Deployment

This module is typically bootstrapped using:

```bash
./devops-tools/tooling/bootstrap
```

This script should be run **after** storage, network, and vault are deployed, as ArgoCD is the final piece that manages everything else.
