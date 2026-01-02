# Cert-Manager

This directory contains the cert-manager deployment configuration for automated TLS certificate management using Cloudflare DNS-01 challenge.

## Our Deployment Approach

We deploy cert-manager using a **dual Helm chart pattern** with Kustomize:

1. **Official cert-manager Helm chart** - The base cert-manager installation
2. **Local wrapper Helm chart** - Our custom resources (issuers, certificates, secrets)

See [kustomization.yaml](./kustomization.yaml):
```yaml
helmCharts:
- name: cert-manager              # Official chart
  repo: https://charts.jetstack.io
  version: v1.19.2
  namespace: network
  valuesFile: cert-manager-values.yaml
  
- name: certmanager               # Our local chart
  namespace: network
  valuesFile: values.yaml          # Values with Vault placeholders
```

This approach allows us to:
- Deploy the official cert-manager with custom resource configurations
- Use Vault for secrets (API tokens, email) via ArgoCD's Vault plugin
- Keep certificate resources in sync with the installation

## Wildcard Certificate Strategy

Instead of requesting individual certificates for each service, we issue a **single wildcard certificate** for the entire domain.

### Why Wildcard?

1. **Simplicity**: One certificate covers all subdomains (`*.yourdomain.com`)
2. **Rate limits**: Avoid Let's Encrypt's per-domain rate limits
3. **Replication**: Share the certificate across all namespaces using Replicator
4. **Centralized management**: One certificate to monitor and renew

### The Certificate

See [templates/certificate.yaml](./templates/certificate.yaml):
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: domain
  namespace: network
spec:
  secretName: domain-tls-secret
  commonName: {{ .Values.domain }}
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  dnsNames:
  - {{ .Values.domain }}              # yourdomain.com
  - "*.{{ .Values.domain }}"          # *.yourdomain.com
  secretTemplate:
    annotations:
      # Replicator annotations for cross-namespace sharing
      replicator.v1.mittwald.de/replication-allowed: "true"
      replicator.v1.mittwald.de/replication-allowed-namespaces: "*"
      replicator.v1.mittwald.de/replicate-to: "*"
```

**Flow:**
1. Cert-manager requests a wildcard cert from Let's Encrypt
2. Certificate is stored as `domain-tls-secret` in the `network` namespace
3. Replicator automatically copies the secret to all namespaces
4. All services can reference `domain-tls-secret` for TLS

## DNS-01 Challenge with Cloudflare

We use **DNS-01 challenge** instead of HTTP-01 because:
- ✅ Required for wildcard certificates
- ✅ Works without exposing port 80 externally
- ✅ No ingress routing needed during challenge
- ✅ Works even if services are down

### ClusterIssuers

We configure two issuers in [templates/issuer.yaml](./templates/issuer.yaml):

**Staging (for testing):**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: {{ .Values.letsEncryptEmail }}
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cert-manager-secrets
            key: api-token
```

**Production (for real certs):**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: {{ .Values.letsEncryptEmail }}
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cert-manager-secrets
            key: api-token
```

**Challenge flow:**
1. Cert-manager creates a TXT record: `_acme-challenge.yourdomain.com`
2. Let's Encrypt queries DNS to verify the record
3. Certificate is issued once verification succeeds
4. TXT record is automatically cleaned up

### Cloudflare API Token

The Cloudflare API token is stored in [templates/secret.yaml](./templates/secret.yaml):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cert-manager-secrets
  namespace: network
type: Opaque
stringData:
  api-token: {{ .Values.apiToken }}
```

The `apiToken` value comes from [values.yaml](./values.yaml):
```yaml
apiToken: <path:secret/data/network/certmanager#apiToken>
```

This Vault placeholder is resolved by ArgoCD's Vault plugin during deployment.

**Required Cloudflare token permissions:**
- Zone - DNS - Edit
- Zone - Zone - Read

## Traefik Default Certificate

We configure Traefik to use our wildcard certificate as the **default certificate** for all HTTPS connections.

See [templates/tlsstore.yaml](./templates/tlsstore.yaml):
```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata:
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
