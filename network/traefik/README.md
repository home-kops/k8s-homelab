# Traefik

Traefik is a modern reverse proxy and ingress controller that routes incoming traffic to your applications.

## What is Traefik?

Traefik acts as the front door to your Kubernetes cluster. When someone visits `https://jellyfin.yourdomain.com`, Traefik receives that request and forwards it to the Jellyfin service inside your cluster.

## What it does

- **HTTP/HTTPS Routing**: Routes requests to services based on domain names and paths
- **TLS Termination**: Handles HTTPS certificates (provided by cert-manager)
- **Load Balancing**: Distributes traffic across multiple pod replicas
- **Automatic Service Discovery**: Automatically finds new services in Kubernetes
- **Middleware**: Add authentication, rate limiting, redirects, etc.
- **Dashboard**: Web UI to monitor traffic and routing rules

## How it works

1. External request arrives at your cluster's IP (provided by MetalLB)
2. Traefik receives the request
3. Traefik checks the request's hostname/path against routing rules (IngressRoutes)
4. Traefik forwards the request to the matching service
5. The service responds through Traefik back to the client

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
- Entry points (ports: 80 for HTTP, 443 for HTTPS)
- TLS configuration
- Dashboard access
- Resource limits
- Integration with CrowdSec bouncer

## Creating Routes

Applications define IngressRoute resources to expose themselves:

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
spec:
  entryPoints:
    - websecure  # HTTPS
  routes:
    - match: Host(`myapp.example.com`)
      kind: Rule
      services:
        - name: my-app-service
          port: 8080
  tls:
    certResolver: letsencrypt  # Automatic certificate
```

## Middleware

Add features to routes using middleware:

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
spec:
  basicAuth:
    secret: auth-secret

---
# Reference middleware in IngressRoute
spec:
  routes:
    - match: Host(`secure.example.com`)
      middlewares:
        - name: basic-auth
      services:
        - name: my-service
          port: 80
```

## Common Middleware

- **basicAuth**: Password protection
- **redirectScheme**: HTTP to HTTPS redirect
- **rateLimit**: Limit requests per IP
- **ipWhiteList**: Restrict access by IP
- **headers**: Modify request/response headers

## Accessing the Dashboard

Traefik's dashboard shows all routes and traffic statistics. Configure access in `values.yaml` or create an IngressRoute:

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.example.com`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
```

## Deployment

Traefik is deployed via the network bootstrap script:

```bash
./network/tooling/bootstrap
```

## Integration with CrowdSec

Traefik includes a CrowdSec bouncer that blocks malicious IPs. Configuration in `templates/bouncer.yaml`.

## Troubleshooting

**503 Service Unavailable**: Service doesn't exist or has no healthy pods

**404 Not Found**: No IngressRoute matches the hostname/path

**Certificate errors**: Check cert-manager logs, verify DNS points to cluster

**Dashboard not accessible**: Verify IngressRoute and authentication configuration
