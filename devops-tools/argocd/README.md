# ArgoCD

This directory contains the ArgoCD deployment configuration for managing the homelab cluster using GitOps principles.

## Our Deployment Approach

Instead of using standard Helm deployment, we use **Kustomize with Helm chart integration** to have better control over the installation. This approach allows us to:

1. Deploy the ArgoCD Helm chart through Kustomize
2. Apply additional manifests alongside the Helm chart (secrets, applications)
3. Use Kustomize's native features for patching and configuration

See [kustomization.yaml](./kustomization.yaml):
```yaml
helmCharts:
- name: argo-cd
  repo: https://argoproj.github.io/argo-helm
  version: 9.2.3
  releaseName: argocd
  namespace: devops-tools
  valuesFile: values.yaml

resources:
- secret.yaml        # Vault plugin credentials
- application.yaml   # Bootstrap application
```

This kustomization deploys three components:
- The ArgoCD Helm chart with custom values
- A sealed secret for Vault authentication
- An Application resource pointing to the central ArgoCD configuration repository

## Vault Plugin Integration

### Why We Need It

All secrets in this homelab are stored in HashiCorp Vault, not directly in Git. ArgoCD needs to resolve Vault placeholders like `<path:secret/data/infra#domain>` when deploying applications. This is where the **ArgoCD Vault Plugin** comes in.

### How It Works

We deploy ArgoCD with a **sidecar container** that extends the repo-server to support Vault secret resolution:

```yaml
# From values.yaml
repoServer:
  # Install the plugin binary
  initContainers:
  - name: download-tools
    image: alpine:3.23
    command: [sh, -c]
    env:
    - name: AVP_VERSION
      value: "1.18.1"
    args:
      - |
        wget -O /custom-tools/argocd-vault-plugin https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v$(AVP_VERSION)/argocd-vault-plugin_$(AVP_VERSION)_linux_amd64
        chmod +x /custom-tools/argocd-vault-plugin
    volumeMounts:
    - mountPath: /custom-tools
      name: custom-tools
  
  # Run the plugin as a sidecar
  extraContainers:
  - name: avp-kustomize
    command: [/var/run/argocd/argocd-cmp-server]
    image: quay.io/argoproj/argocd
    securityContext:
      runAsNonRoot: true
      runAsUser: 999
```

**Plugin Flow:**
1. ArgoCD detects a kustomization.yaml in a repository
2. Instead of running `kustomize build` directly, it uses our custom plugin
3. The plugin runs: `kustomize build . --enable-helm | argocd-vault-plugin generate -`
4. Vault placeholders are replaced with actual secrets from Vault
5. The final manifests are applied to the cluster

### Vault Authentication

The plugin needs to authenticate to access the Vault. Follow [this guide](https://argocd-vault-plugin.readthedocs.io/en/stable/config/) to allow ArgoCD to access the Vault.

### Plugin Configuration

The Config Management Plugin (CMP) is defined in the ArgoCD configuration:

```yaml
# From values.yaml
configs:
  cmp:
    create: true
    plugins:
      argocd-vault-plugin-kustomize:
        discover:
          find:
            command:
            - sh
            - "-c"
            - "find . -name kustomization.yaml"
        generate:
          command:
          - bash
          - "-c"
          - |
            kustomize build . --enable-helm | argocd-vault-plugin generate --secret-name devops-tools:argocd-vault-plugin-credentials -
```

This tells ArgoCD:
- **Discover**: Use this plugin for any repository with a `kustomization.yaml` file
- **Generate**: Run kustomize, pipe output through the Vault plugin to resolve secrets

## Bootstrap Application

After ArgoCD is deployed, it immediately syncs an Application pointing to the central configuration repository:

```yaml
# From application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-config
spec:
  source:
    repoURL: https://github.com/home-kops/argocd-config.git
    targetRevision: main
    path: .
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This repository contains all other Application definitions for the cluster. It's the "app of apps" pattern.

## Repository Credentials

GitHub credentials are configured in ArgoCD to access private repositories:

```yaml
# From values.yaml
configs:
  credentialTemplates:
    home-kops-credentials:
      url: https://github.com/home-kops/
      password: <path:secret/data/devopstools/argocd#gh_org1_password>
      username: <path:secret/data/devopstools/argocd#gh_org1_username>
```

These credentials are also resolved from Vault during deployment.

## Accessing the UI

Once deployed, access ArgoCD through the configured ingress:

```yaml
# From values.yaml
server:
  ingress:
    enabled: true
    ingressClassName: traefik
    annotations:
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
    tls: true

global:
  domain: cd.<path:secret/data/infra#domain>
```

The URL will be: `https://cd.yourdomain.com`

Admin credentials:
- Username: `admin`
- Password: Retrieved from Vault via `<path:secret/data/devopstools/argocd#admin_password>`

## Deployment

Deploy ArgoCD using the bootstrap script:

```bash
./devops-tools/tooling/bootstrap
```

This should be the **last** bootstrap step since ArgoCD manages all other applications after this point.

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
