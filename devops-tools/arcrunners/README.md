# GitHub Arc Runners

Self-hosted GitHub Actions runners executing CI/CD workflows in the cluster.

## Configuration

### Runner Group

```yaml
runnerGroup: "homelab"
```

Reference in workflows:
```yaml
runs-on: [self-hosted, homelab]
```

## Resources

Default Helm chart resource allocations.

## Dependencies

- **Controller**: `gha-scale-set-controller` (manages runner lifecycle)

## References

- [Actions Runner Controller Documentation](https://github.com/actions/actions-runner-controller)
- [GitHub Actions Self-hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners)

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
