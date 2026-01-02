# Traefik

Ingress controller routing external HTTP/HTTPS traffic to cluster services.

## Deployment

**Manually bootstrapped** via [tooling/bootstrap](../tooling/bootstrap) script.

Dual Helm chart pattern:
1. Official Traefik chart ([traefik-values.yaml](./traefik-values.yaml))
2. Local templates for CrowdSec bouncer ([templates/](./templates/))

## Configuration

### Domain

Cluster domain from Vault:
```yaml
domain: <path:secret/data/infra#k8s_domain>
```

### CrowdSec Bouncer Middleware

Protects ingress routes ([templates/bouncer.yaml](./templates/bouncer.yaml)):
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: crowdsec-bouncer
spec:
  plugin:
    bouncer:
      crowdsecLapiKey: <path:secret/data/monitoring/crowdsec#traefik_bouncer_key>
```

Apply to IngressRoutes:
```yaml
middlewares:
  - name: crowdsec-bouncer
```

### LoadBalancer Service

Receives IP `192.168.10.99` from MetalLB.

## Resources

- **CPU**: 100m requests, 300m limits
- **Memory**: 50Mi requests, 150Mi limits

## Access

External traffic → `192.168.10.99:80/443` → Traefik → Services

## References

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Helm Chart](https://github.com/traefik/traefik-helm-chart)

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
