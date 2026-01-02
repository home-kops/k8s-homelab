# Network

This module contains the core networking infrastructure that makes your applications accessible both inside and outside the cluster.

## What's included?

### [Traefik](./traefik/README.md)
A modern reverse proxy and ingress controller that routes external traffic to your applications.

**What it does**: Acts as the front door to your cluster. Routes HTTP/HTTPS requests to the correct application based on domain names and paths.

### [Cert-Manager](./certmanager/README.md)
Automated certificate management for TLS/SSL.

**What it does**: Automatically obtains, renews, and manages SSL certificates from Let's Encrypt, so your applications are accessible over HTTPS.

### [MetalLB](./metallb/README.md)
A load balancer implementation for bare metal Kubernetes clusters.

**What it does**: Provides LoadBalancer-type services in clusters that don't run on a cloud provider. Assigns external IPs from a configured pool.

### [Replicator](./replicator/README.md)
Automatically replicates secrets and configmaps across namespaces.

**What it does**: Copies specific secrets/configmaps to multiple namespaces, useful for sharing TLS certificates or common configurations.

## How they work together

These components form the networking foundation of your cluster:

1. **MetalLB** assigns an external IP to Traefik's LoadBalancer service
2. **Traefik** receives incoming HTTP/HTTPS requests on that IP
3. **Cert-Manager** provides TLS certificates to Traefik for HTTPS
4. **Traefik** routes requests to the appropriate service based on domain/path rules
5. **Replicator** ensures shared secrets (like wildcard TLS certs) are available in all namespaces

## Deployment Order

Network infrastructure must be deployed early because other services depend on it:

```bash
./network/tooling/bootstrap
```

This should be run **after** storage but **before** most other services.

## Configuration

- **MetalLB IP Pool**: Configure in [metallb/ip-addresspool.yaml](./metallb/ip-addresspool.yaml)
- **Traefik Settings**: Configure in [traefik/values.yaml](./traefik/values.yaml)
- **Cert-Manager Issuers**: Configure in [certmanager/templates/issuer.yaml](./certmanager/templates/issuer.yaml)

## Accessing Applications

Once deployed, applications expose IngressRoute resources (Traefik-specific) to define routing rules. Example:

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`myapp.example.com`)
      kind: Rule
      services:
        - name: my-app-service
          port: 80
  tls:
    certResolver: letsencrypt
```
