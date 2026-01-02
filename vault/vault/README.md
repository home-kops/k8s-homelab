# HashiCorp Vault

Centralized secrets management system for securely storing and accessing sensitive data.

## What is Vault?

Vault is a secure storage system for secrets like passwords, API keys, and certificates. Instead of hardcoding secrets in your manifests or storing them as plain text, Vault encrypts and manages them centrally.

## What it does

- **Secure storage**: Encrypts secrets at rest
- **Dynamic secrets**: Generate temporary credentials on-demand
- **Access control**: Fine-grained policies for who can access what
- **Audit logging**: Track all secret access
- **Secret rotation**: Automatically rotate credentials
- **Multiple backends**: Support for databases, cloud providers, SSH, etc.

## How it works

1. Vault starts in a "sealed" state (encrypted and inaccessible)
2. It must be "unsealed" using unseal keys (handled automatically by vault-unsealer)
3. Applications authenticate to Vault (using Kubernetes service accounts)
4. Applications request secrets they need
5. Vault checks policies to ensure the app is authorized
6. Vault returns the secret
7. All access is logged for audit

## Architecture

- **Sealed/Unsealed**: Vault starts sealed, must be unsealed to access secrets
- **Storage backend**: File storage (in this setup), could be etcd, Consul, etc.
- **Unseal keys**: Required to decrypt the vault (stored securely by unsealer)
- **Root token**: Master key for initial configuration

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
server:
  standalone:
    config: |
      storage "file" {
        path = "/vault/data"
      }
      listener "tcp" {
        address     = "0.0.0.0:8200"
        tls_disable = 1
      }
  dataStorage:
    size: 8Gi
    storageClass: nfs-sc
```

## Initial Setup

After deployment, Vault needs to be initialized:

1. **Initialize Vault** (first time only):
   ```bash
   kubectl exec -n vault vault-0 -- vault operator init
   ```
   
   This outputs:
   - 5 unseal keys (store securely!)
   - Root token (store securely!)

2. **Configure authentication**:
   ```bash
   ./vault/tooling/configure
   ```
   
   This sets up Kubernetes authentication so pods can access secrets.

3. **Load secrets**:
   ```bash
   ./vault/tooling/load-secrets
   ```
   
   This loads your secrets into Vault.

## Using Secrets in Applications

Use the Vault path notation in your manifests:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  database-password: <path:secret/data/myapp#db_password>
  api-key: <path:secret/data/myapp#api_key>
```

When ArgoCD deploys this, it replaces the placeholders with actual values from Vault.

## Secret Paths

Secrets are organized in paths:

```
secret/
  └── data/
      ├── infra/           # Infrastructure secrets
      │   ├── domain
      │   └── email
      ├── monitoring/      # Monitoring secrets
      │   └── crowdsec/
      │       └── traefik_bouncer_key
      └── media/           # Media app secrets
          ├── jellyfin/
          └── calibre/
```

Access a secret: `<path:secret/data/monitoring/crowdsec#traefik_bouncer_key>`

## Creating Secrets

### Via UI
1. Access Vault UI at `http://vault.yourdomain.com:8200`
2. Login with root token
3. Navigate to secret engine
4. Create key-value pairs

### Via CLI
```bash
kubectl exec -n vault vault-0 -- vault kv put secret/myapp \
  db_password=supersecret \
  api_key=abc123
```

### Via Script
```bash
./vault/tooling/load-secrets
```

## Policies

Control access with policies:

```hcl
# Allow reading app secrets
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
```

## Authentication Methods

- **Token**: Direct token authentication
- **Kubernetes**: Authenticate using Kubernetes service account (recommended for pods)
- **AppRole**: Application-based authentication
- **LDAP/AD**: Enterprise user authentication

## Deployment

Deploy via the vault bootstrap script:

```bash
./vault/tooling/bootstrap
```

This deploys both Vault and the vault-unsealer.

## High Availability Notes

This setup uses standalone mode (single instance). For production:
- Use HA mode with 3+ replicas
- Use a distributed storage backend (Consul, etcd)
- Store unseal keys in multiple secure locations

## Backup and Recovery

**Critical**: Back up Vault's storage regularly!

```bash
# Backup the storage PV
kubectl exec -n vault vault-0 -- tar czf /tmp/vault-backup.tar.gz /vault/data

# Copy out of pod
kubectl cp vault/vault-0:/tmp/vault-backup.tar.gz ./vault-backup.tar.gz
```

Store backups securely - they contain encrypted secrets, but unseal keys could decrypt them.

## Troubleshooting

**Vault is sealed**: Check vault-unsealer logs, ensure unseal keys are correct

**Can't access secrets**: Check authentication, verify policies grant access

**Permission denied**: Review Vault policies, ensure service account has correct role

**Performance issues**: Vault is lightweight, but check storage I/O if slow

## Security Best Practices

- Never commit root token or unseal keys to Git
- Rotate root token after initial setup
- Use least-privilege policies
- Enable audit logging
- Regularly rotate secrets
- Use dynamic secrets where possible
- Monitor audit logs for suspicious access
