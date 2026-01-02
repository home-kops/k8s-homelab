# Vault

This module provides centralized secrets management using HashiCorp Vault.

## What's included?

### [Vault](./vault/README.md)
HashiCorp Vault server for storing and managing sensitive data.

**What it does**: Securely stores secrets (passwords, API keys, certificates) and provides them to applications when needed.

### [Vault Unsealer](./unsealer/README.md)
Automated unsealing service for Vault.

**What it does**: Automatically unseals Vault when it starts or restarts, so you don't have to manually enter unseal keys.

## Why you need this

Hardcoding secrets in your manifests is dangerous. Vault provides:

- **Encryption at rest**: Secrets are encrypted in storage
- **Audit logging**: Track who accessed what secrets and when
- **Dynamic secrets**: Generate temporary credentials on-demand
- **Centralized management**: One place to manage all secrets

## How it works

1. **Vault** stores secrets encrypted in its backend storage
2. When Vault starts, it's "sealed" (encrypted and inaccessible)
3. **Vault Unsealer** automatically unseals it using stored unseal keys
4. Applications authenticate to Vault and retrieve their secrets
5. Applications use special syntax in their manifests: `<path:secret/data/path#key>`

These placeholders are replaced with actual secret values by ArgoCD's Vault plugin.

## Deployment

Vault must be deployed **after** storage and network, but **before** applications that need secrets:

```bash
./vault/tooling/bootstrap
```

## Configuration and Unsealing

After initial deployment, Vault needs to be initialized. The `vault/tooling/` directory contains helper scripts:

- `bootstrap`: Deploys Vault and unsealer
- `configure`: Configures Vault policies and authentication
- `load-secrets`: Loads secrets into Vault
- `load-secret-files`: Loads files as secrets

## Using Secrets in Applications

Reference Vault secrets in your manifests using the path notation:

```yaml
env:
  - name: DATABASE_PASSWORD
    value: <path:secret/data/myapp#db_password>
```

When ArgoCD deploys this, it replaces the placeholder with the actual secret from Vault.

## Security Notes

- Unseal keys should be stored securely (the unsealer automates this)
- Use Vault's AppRole or Kubernetes authentication for applications
- Regularly rotate secrets stored in Vault
- Enable audit logging to track secret access
