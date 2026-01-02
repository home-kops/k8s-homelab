# ArgoCD

GitOps continuous delivery tool managing the homelab cluster.

## Deployment

**Manually bootstrapped** via [tooling/bootstrap](../tooling/bootstrap) script.

Kustomize with Helm chart integration ([kustomization.yaml](./kustomization.yaml)):

```yaml
helmCharts:
- name: argo-cd
  repo: https://argoproj.github.io/argo-helm
  valuesFile: values.yaml

resources:
- secret.yaml
- application.yaml
```

## Configuration

### ArgoCD Vault Plugin (AVP)

Sidecar pattern for Vault secret resolution:

1. Init container downloads argocd-vault-plugin binary
2. Sidecar container runs as Config Management Plugin
3. Plugin workflow: `kustomize build --enable-helm | argocd-vault-plugin generate -`
4. Replaces `<path:secret/data/path#key>` with Vault values

Configuration in [values.yaml](./values.yaml).

### Vault Authentication

Kubernetes auth method using ServiceAccount token.

### Bootstrap Application

[application.yaml](./application.yaml) creates initial Application pointing to central config repository.

## Resources

Default Helm chart allocations.

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Vault Plugin](https://argocd-vault-plugin.readthedocs.io/)
- [Helm Chart](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd)

## Accessing the UI

`https://cd.yourdomain.com`

Admin credentials:
- Username: `admin`
- Password: Retrieved from Vault via `<path:secret/data/devopstools/argocd#admin_password>`

## Troubleshooting

**Plugin not working**: 
- Check repo-server logs: `kubectl logs -n devops-tools -l app.kubernetes.io/component=repo-server -c avp-kustomize`
- Verify Vault is accessible from the cluster
- Check the sealed secret was properly decrypted

**Vault authentication failing**:
- Ensure Vault's Kubernetes auth is configured for the `argocd` role
- Verify repo-server service account has proper RBAC permissions
- Check `VAULT_ADDR` in the plugin credentials secret

**Application won't sync**:
- Verify repository credentials are correct
- Check if repository contains `kustomization.yaml`
- Review application logs in the ArgoCD UI

**Secrets not resolved**:
- Confirm secret exists in Vault at the specified path
- Check Vault token has read permissions for that path
- Verify placeholder syntax: `<path:secret/data/path#key>`
