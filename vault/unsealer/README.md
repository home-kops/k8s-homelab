# Vault Unsealer

Automated service that unseals HashiCorp Vault after restarts or reboots.

## What is the Vault Unsealer?

When Vault starts, it's "sealed" - encrypted and inaccessible. To use Vault, it must be "unsealed" using unseal keys. The vault-unsealer automates this process so you don't have to manually unseal Vault every time it restarts.

## Why is this needed?

Vault seals itself for security:
- When Vault starts
- After a pod restart
- After a node reboot
- After any interruption

Unsealing requires entering 3 out of 5 unseal keys (by default). Doing this manually every time is impractical, so the unsealer automates it.

## What it does

- **Monitors Vault**: Checks if Vault is sealed
- **Automatic unsealing**: Unseals Vault using stored unseal keys
- **Runs on schedule**: CronJob that runs periodically (e.g., every 5 minutes)
- **Handles restarts**: Ensures Vault is always available

## How it works

1. CronJob triggers every few minutes
2. Unsealer checks Vault's seal status
3. If sealed, it retrieves unseal keys from a Kubernetes secret
4. It calls Vault's unseal API with the keys
5. Vault unseals and becomes operational

## Security Considerations

The unsealer stores unseal keys as a Kubernetes secret. This is a trade-off:
- **Pro**: Vault automatically unseals, improving availability
- **Con**: Unseal keys are accessible in the cluster

For maximum security (less availability):
- Don't use the unsealer
- Manually unseal Vault when needed
- Store unseal keys in a hardware security module (HSM)

For homelab (more availability):
- Use the unsealer with encrypted secrets
- Secure cluster access with RBAC
- Regularly rotate unseal keys

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
cronjob:
  schedule: "*/5 * * * *"    # Run every 5 minutes
  image:
    repository: vault
    tag: latest
```

The unseal keys secret: [templates/secret.yaml](./templates/secret.yaml)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-unseal-keys
type: Opaque
stringData:
  key1: <path:secret/data/vault/unseal-keys#key1>
  key2: <path:secret/data/vault/unseal-keys#key2>
  key3: <path:secret/data/vault/unseal-keys#key3>
```

## Initial Setup

After initializing Vault for the first time:

1. **Initialize Vault** (if not already done):
   ```bash
   kubectl exec -n vault vault-0 -- vault operator init
   ```
   
   Save the 5 unseal keys and root token securely.

2. **Store unseal keys in Vault** (circular dependency solved by manual first unseal):
   ```bash
   kubectl exec -n vault vault-0 -- vault kv put secret/vault/unseal-keys \
     key1=<unseal-key-1> \
     key2=<unseal-key-2> \
     key3=<unseal-key-3>
   ```

3. **Deploy unsealer**: ArgoCD deploys it, or manually:
   ```bash
   kubectl apply -k vault/unsealer/
   ```

## How Unsealing Works

Vault uses Shamir's Secret Sharing:
- Vault generates 5 unseal keys during initialization
- Any 3 of the 5 keys can unseal Vault (threshold)
- This prevents any single person from accessing Vault

The unsealer stores 3+ keys and automatically provides them when needed.

## Monitoring

Check if Vault is sealed:
```bash
kubectl exec -n vault vault-0 -- vault status
```

Output:
```
Sealed: false  # Good - Vault is operational
```

Check unsealer logs:
```bash
kubectl logs -n vault -l app=vault-unsealer
```

## Troubleshooting

**Vault remains sealed**: Check unsealer logs, verify secret contains correct keys

**CronJob not running**: Check CronJob definition, ensure schedule is correct

**Insufficient keys error**: Need at least 3 valid unseal keys

**Unsealer can't reach Vault**: Network policy issue or Vault service not available

## Alternative Solutions

Instead of vault-unsealer, you could:
- **Manual unsealing**: More secure, less available
- **Cloud KMS auto-unseal**: Use AWS KMS, Azure Key Vault, etc. (requires cloud integration)
- **HSM auto-unseal**: Hardware security module (expensive, enterprise-grade)

## Deployment

Deployed automatically with Vault via the bootstrap script:

```bash
./vault/tooling/bootstrap
```

The unsealer starts working immediately after Vault is initialized.

## Best Practices

- **Backup unseal keys**: Store the 5 original keys securely offline
- **Test unsealing**: Periodically test manual unsealing with backup keys
- **Monitor logs**: Watch unsealer logs for failures
- **Rotate keys**: Periodically rekey Vault to generate new unseal keys
- **Secure secret**: Use RBAC to restrict access to the unseal-keys secret
