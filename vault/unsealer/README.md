# Vault Unsealer

Automated service unsealing HashiCorp Vault after restarts.

## Deployment

**Manually bootstrapped** via [tooling/bootstrap](../tooling/bootstrap) script.

Local Helm chart as CronJob:
- [cronjob.yaml](./templates/cronjob.yaml)
- [secret.yaml](./templates/secret.yaml)

## Configuration

### Vault Connection

```yaml
vault:
  server: http://vault:8200
```

### Unseal Keys

Three unseal keys from Vault:
```yaml
vault:
  key1: <path:secret/data/vault/unsealer#key1>
  key2: <path:secret/data/vault/unsealer#key2>
  key3: <path:secret/data/vault/unsealer#key3>
```

### Container Image

[hashicorp/vault](https://hub.docker.com/r/hashicorp/vault)

## Resources

Default allocations.

## Dependencies

- Vault

## References

- [Vault Seal/Unseal Documentation](https://developer.hashicorp.com/vault/docs/concepts/seal)
- [Vault CLI](https://developer.hashicorp.com/vault/docs/commands)

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
