# Renovate

Automated dependency update tool managing container images and Helm charts.

## Configuration

### Execution Schedule

**Hourly CronJob**:
```yaml
mendRnvCronJobSchedulerAll: '0 * * * *'
```
Checks for updates every hour at minute 0.

### Kubernetes File Scanning

Scans **all YAML files** for container images and Helm chart versions:
```yaml
config: |
  module.exports = {
    "kubernetes": {
      "fileMatch": ["\\.yaml$"]
    }
  }
```

Detects images in Deployments, StatefulSets, DaemonSets, init containers, and sidecars.

## Resources

- **Memory**: 4Gi limit (configured in values.yaml)
- Other resources: Default Helm chart values

## Dependencies

Requires GitHub App installation on target repositories.

## References

- [Renovate Documentation](https://docs.renovatebot.com/)
- [Mend Renovate CE](https://github.com/mend/renovate-ce-ee)
- [Helm Chart](https://mend.github.io/renovate-ce-ee/)
- Update policies

## Web Interface

`https://renovate.yourdomain.com`

The dashboard shows:
- Repository sync status
- Recent job runs
- Pull request statistics
- Error logs

## Resource Allocation

Renovate is configured with specific resource limits and runs on small-sized nodes:

```yaml
# From values.yaml
resources:
  requests:
    cpu: 250m
    memory: 2Gi
  limits:
    cpu: 1000m
    memory: 4Gi

nodeSelector:
  size: s
```

Renovate can be resource-intensive when scanning large repositories with many dependencies. The 4Gi memory limit prevents OOM issues during scanning.

## Deployment

Renovate is deployed by ArgoCD after the initial DevOps tools bootstrap.

## Troubleshooting

**No PRs being created:**
- Check CronJob is running: `kubectl get cronjob -n devops-tools`
- Review Renovate logs for errors
- Verify GitHub App has correct permissions
- Check repository is detected in Renovate dashboard

**Authentication errors:**
- Verify GitHub App is installed on the repository
- Check app ID and private key in Vault are correct
- Ensure token has not expired

**High memory usage:**
- Increase memory limits in values.yaml
- Reduce scan frequency if scanning many large repositories

**Updates not merged:**
- Renovate creates PRs but doesn't auto-merge by default
- Configure auto-merge rules in renovate.json if desired
- PRs must be manually reviewed and merged
