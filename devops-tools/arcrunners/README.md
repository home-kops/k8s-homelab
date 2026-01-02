# GitHub Arc Runners

Self-hosted GitHub Actions runners that execute your CI/CD workflows directly in your Kubernetes cluster.

## What are Arc Runners?

Arc (Actions Runner Controller) runs GitHub Actions workflows on your own infrastructure instead of GitHub's cloud runners. This is useful when you need:
- Access to local resources (databases, services in your cluster)
- Custom hardware (GPUs, specific CPU architectures)
- Reduced costs for high-volume CI/CD
- Faster builds (no cold start delays)

## What they do

- **Run GitHub Actions**: Execute workflows defined in `.github/workflows/`
- **Scale automatically**: Spin up runners on-demand when workflows trigger
- **Clean environment**: Each workflow gets a fresh container
- **Resource control**: Set CPU/memory limits for runners

## How it works

1. A GitHub Actions workflow is triggered (push, PR, schedule, etc.)
2. GitHub sends the job to your Arc runner pool
3. Arc creates a new pod in your cluster
4. The workflow executes in that pod
5. Results are sent back to GitHub
6. The pod is deleted

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
- GitHub App credentials (for authentication)
- Runner groups and labels
- Resource limits (CPU, memory)
- Node selector (which nodes can run workflows)

## Setting up GitHub Integration

You need to register the runners with GitHub:

1. Create a GitHub App or use a Personal Access Token
2. Configure the credentials in `values.yaml`
3. Specify which repositories can use these runners

## Using in Workflows

Reference your self-hosted runners in workflows:

```yaml
name: Build
on: [push]
jobs:
  build:
    runs-on: self-hosted  # Use your Arc runners
    steps:
      - uses: actions/checkout@v3
      - run: echo "Running on my homelab!"
```

Or use specific labels:

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64]
```

## Deployment

Arc Runners are deployed by ArgoCD after the DevOps tools bootstrap.

## Security Considerations

- **Isolation**: Runners should have limited cluster permissions
- **Trusted workflows only**: Only run workflows from repositories you control
- **Resource limits**: Set appropriate CPU/memory limits
- **Network policies**: Restrict what runners can access

## Troubleshooting

**Workflows not starting**: Check runner registration with GitHub, verify credentials

**Runners failing**: Check pod logs, ensure necessary tools are installed in runner image

**Resource exhaustion**: Adjust resource limits or node selector to spread load
