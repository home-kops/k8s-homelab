# Cert-Manager

Automated TLS certificate management using Let's Encrypt and Cloudflare DNS-01 challenge.

## Deployment

**Manually bootstrapped** via [tooling/bootstrap](../tooling/bootstrap) script.

Dual Helm chart pattern:
1. Official cert-manager chart ([cert-manager-values.yaml](./cert-manager-values.yaml))
2. Local chart with custom resources ([templates/](./templates/))

## Configuration

### Wildcard Certificate

Single certificate for entire domain:
```yaml
dnsNames:
  - {{ .Values.domain }}
  - "*.{{ .Values.domain }}"
```

Stored as `domain-tls-secret` in `network` namespace.

Replicated to all namespaces:
```yaml
secretTemplate:
  annotations:
    replicator.v1.mittwald.de/replicate-to: "*"
```

### Cloudflare DNS-01

API token from Vault:
```yaml
cloudflareApiToken: <path:secret/data/infra#cloudflare_api_token>
```

### ClusterIssuers

Two Let's Encrypt issuers:
- `letsencrypt-staging`
- `letsencrypt-production`

Email from Vault:
```yaml
letsEncryptEmail: <path:secret/data/infra#letsencrypt_email>
```

### Traefik Default Certificate

TLSStore configures Traefik default:
```yaml
defaultCertificate:
  secretName: domain-tls-secret
```

## Resources

Default Helm chart allocations.

## Dependencies

- Replicator
- Cloudflare DNS

## References

- [Cert-Manager Documentation](https://cert-manager.io/docs/)
- [Helm Chart](https://github.com/cert-manager/cert-manager/tree/master/deploy/charts/cert-manager)
  name: default
  namespace: default
spec:
  defaultCertificate:
    secretName: domain-tls-secret
```

This TLSStore resource tells Traefik:
- Use `domain-tls-secret` as the default certificate
- Apply it to any HTTPS connection that doesn't specify a certificate
- Must be in the `default` namespace (Traefik requirement)

**Result:** All IngressRoutes automatically get HTTPS without specifying TLS configuration.

## Configuration Values

All configuration is templated through [values.yaml](./values.yaml):

```yaml
domain: <path:secret/data/infra#domain>
letsEncryptEmail: <path:secret/data/network/certmanager#letsEncryptEmail>
apiToken: <path:secret/data/network/certmanager#apiToken>
```

These values are:
1. Stored in Vault
2. Resolved by ArgoCD's Vault plugin during sync
3. Templated into all resources

**Vault paths:**
- `secret/data/infra#domain` - Your domain name
- `secret/data/network/certmanager#letsEncryptEmail` - Email for Let's Encrypt notifications
- `secret/data/network/certmanager#apiToken` - Cloudflare API token

## Resource Management

Cert-manager components run on control plane nodes with specific resource limits:

```yaml
# From cert-manager-values.yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

nodeSelector:
  role: cp

tolerations:
- key: "role"
  operator: "Equal"
  value: "cp"
  effect: "NoSchedule"
```

All three components (cert-manager, webhook, cainjector) are configured to run on control plane nodes.

## Deployment

Deploy cert-manager via the network bootstrap:

```bash
./network/tooling/bootstrap
```

**What happens:**
1. Cert-manager CRDs are installed
2. Cert-manager controller, webhook, and cainjector are deployed
3. ClusterIssuers are created (staging and production)
4. Cloudflare API token secret is created
5. Wildcard certificate is requested
6. Certificate is replicated to all namespaces
7. Traefik is configured to use the certificate as default

## Monitoring Certificate Status

Check the wildcard certificate:
```bash
kubectl get certificate -n network
kubectl describe certificate domain -n network
```

Check certificate requests and challenges:
```bash
kubectl get certificaterequest -n network
kubectl get challenge -n network
```

Check replicated secrets in other namespaces:
```bash
kubectl get secret domain-tls-secret -A
```

View certificate details:
```bash
kubectl get secret domain-tls-secret -n network -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

## Troubleshooting

**Certificate not issuing:**
- Check ClusterIssuer status: `kubectl describe clusterissuer letsencrypt-production`
- Review cert-manager logs: `kubectl logs -n network -l app=cert-manager`
- Check DNS propagation for TXT records: `dig _acme-challenge.yourdomain.com TXT`

**Cloudflare API errors:**
- Verify API token has correct permissions (Zone DNS Edit, Zone Read)
- Check token hasn't expired
- Ensure domain is active in Cloudflare

**Certificate not replicated:**
- Verify Replicator is running: `kubectl get pods -n network -l app=replicator`
- Check Replicator logs for errors
- Verify secret has replication annotations

**Traefik not using certificate:**
- Ensure TLSStore exists in `default` namespace
- Check that `domain-tls-secret` exists in `default` namespace (via Replicator)
- Restart Traefik pods if needed

**Rate limit errors:**
- Use `letsencrypt-staging` issuer for testing
- Let's Encrypt production has limits: 50 certs/week per domain
- Staging has much higher limits
