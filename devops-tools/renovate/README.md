# Renovate

Renovate automatically keeps your dependencies up to date by creating pull requests when new versions are available.

## What is Renovate?

Think of Renovate as your automated maintenance assistant. It regularly checks all your container images and Helm charts, finds when new versions are released, and creates pull requests with the updates.

## What it does

- **Scans manifests**: Finds all container image references and Helm chart versions
- **Checks for updates**: Queries container registries and Helm repositories for newer versions
- **Creates PRs**: Automatically creates pull requests with version updates
- **Smart scheduling**: Can be configured to create PRs on specific days/times
- **Grouping**: Groups related updates together (e.g., all monitoring stack updates)

## How it works

1. Renovate runs on a schedule (e.g., daily)
2. It scans all YAML files in the repository
3. For each container image or Helm chart, it checks for newer versions
4. If updates are found, it creates a pull request with:
   - Updated version numbers
   - Release notes from the new version
   - Links to changelogs
5. You review the PR, test if needed, and merge
6. ArgoCD automatically deploys the updates

## Benefits

- **Never miss security updates**: Get notified immediately about new releases
- **Reduce manual work**: No more manually checking for updates
- **Consistent updates**: All components follow the same update process
- **Visibility**: See all available updates in one place

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
- Schedule: When to check for updates
- Repository configuration: Which repository to monitor
- Auto-merge rules: Automatically merge certain updates (e.g., patch versions)
- Package rules: Custom behavior for specific dependencies

## Example Configuration

```yaml
# renovate.json at repository root
{
  "extends": ["config:base"],
  "kubernetes": {
    "fileMatch": [".*\\.yaml$"]
  },
  "schedule": ["after 10pm every weekday"],
  "automerge": true,
  "automergeType": "pr",
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true
    }
  ]
}
```

## Deployment

Renovate is deployed by ArgoCD after the initial bootstrap.

Ensure your repository is configured in `values.yaml` and credentials for accessing the repository are provided.

## Reviewing Updates

When Renovate creates a PR:
1. Review the changelog/release notes
2. Check if the update requires configuration changes
3. Test in a non-production environment if possible
4. Merge the PR
5. ArgoCD automatically deploys the update

## Troubleshooting

**No PRs being created**: Check Renovate logs, verify repository access credentials

**Too many PRs**: Adjust scheduling or grouping rules in `renovate.json`

**Updates failing**: Some updates may require manual intervention (breaking changes)
