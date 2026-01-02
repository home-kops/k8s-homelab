# MetalLB

Load balancer implementation for bare metal Kubernetes clusters.

## What is MetalLB?

MetalLB provides LoadBalancer-type services in clusters that don't run on cloud providers. In cloud environments (AWS, GCP, Azure), LoadBalancers are provided automatically. On bare metal or homelab clusters, MetalLB fills that gap.

## What it does

- **External IPs**: Assigns real IP addresses to LoadBalancer services
- **Network integration**: Makes services accessible from outside the cluster
- **Two modes**: Layer 2 (ARP) or BGP routing
- **IP pool management**: Manages a pool of available IP addresses

## Why you need it

Without MetalLB on bare metal:
- LoadBalancer services stay "Pending" forever
- You're limited to NodePort (high ports like 30000+)
- No clean way to expose services externally

With MetalLB:
- LoadBalancer services get real external IPs
- Access services on standard ports (80, 443)
- Clean integration with Traefik and other ingress controllers

## How it works

### Layer 2 Mode (Most common for homelabs)

1. You create a LoadBalancer service
2. MetalLB assigns an IP from its pool
3. MetalLB responds to ARP requests for that IP
4. External clients can reach the service via that IP

### BGP Mode (Advanced)

1. MetalLB peers with your network router via BGP
2. It advertises service IPs to your router
3. Router handles routing to the cluster

## Configuration

Main configuration file: [ip-addresspool.yaml](./ip-addresspool.yaml)

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: network
spec:
  addresses:
    - 192.168.1.200-192.168.1.220  # IP range to use
```

### L2Advertisement

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: network
spec:
  ipAddressPools:
    - homelab-pool
```

## IP Pool Planning

Plan your IP range carefully:

**Example home network**: `192.168.1.0/24`
- `192.168.1.1`: Router
- `192.168.1.2-192.168.1.99`: DHCP range (devices)
- `192.168.1.100-192.168.1.199`: Static IPs (servers, printers)
- `192.168.1.200-192.168.1.220`: MetalLB pool ✓
- `192.168.1.221-192.168.1.254`: Reserved

**Important**: 
- Don't overlap with DHCP
- Reserve IPs outside DHCP range
- Document your allocation

## Usage

Once deployed, create LoadBalancer services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
spec:
  type: LoadBalancer  # MetalLB assigns an IP
  ports:
    - port: 80
      targetPort: 8080
      name: web
    - port: 443
      targetPort: 8443
      name: websecure
  selector:
    app: traefik
```

Check assigned IP:
```bash
kubectl get svc traefik -n network
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
traefik   LoadBalancer   10.43.100.50    192.168.1.200    80:30080/TCP,443:30443/TCP
```

## Deployment

Deployed via the network bootstrap:

```bash
./network/tooling/bootstrap
```

## Typical Setup

In most homelabs:
1. MetalLB assigns IP `192.168.1.200` to Traefik
2. Configure DNS: `*.yourdomain.com` → `192.168.1.200`
3. All HTTP/HTTPS traffic goes to that IP
4. Traefik routes requests to appropriate services

## Layer 2 vs BGP

### Layer 2 (ARP)
**Pros**:
- ✅ Simple setup, no router configuration
- ✅ Works on any network
- ✅ No special hardware needed

**Cons**:
- ❌ Single node handles traffic (no load balancing)
- ❌ Failover takes 10 seconds
- ❌ Traffic concentrated on one node

### BGP
**Pros**:
- ✅ True load balancing across nodes
- ✅ Fast failover
- ✅ Scales better

**Cons**:
- ❌ Requires BGP-capable router
- ❌ More complex configuration
- ❌ Overkill for small homelabs

**Recommendation**: Use Layer 2 for homelabs, BGP for production.

## Monitoring

Check MetalLB components:
```bash
kubectl get pods -n metallb-system
kubectl logs -n metallb-system -l app=metallb
```

Check IP assignments:
```bash
kubectl get svc -A | grep LoadBalancer
```

## Troubleshooting

**External IP shows "Pending"**: 
- Check MetalLB pods are running
- Verify IPAddressPool and L2Advertisement exist
- Check IP pool has available addresses

**Can't reach service from outside**:
- Verify IP is in correct subnet
- Check firewall rules
- Ensure IP isn't used by another device
- Test from same subnet first

**IP conflicts**:
- Another device is using an IP from the pool
- Remove IP from MetalLB pool or change device's IP

**Failover not working**:
- Layer 2 failover takes ~10 seconds (normal)
- Check node connectivity

## Advanced Configuration

### Specific IP assignment

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.1.205
spec:
  type: LoadBalancer
```

### Multiple IP pools

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: internal-pool
spec:
  addresses:
    - 192.168.1.200-192.168.1.209
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: dmz-pool
spec:
  addresses:
    - 10.0.0.100-10.0.0.109
```

## DNS Configuration

Point your domain to MetalLB IPs:

```
# Internal DNS or hosts file
*.yourdomain.com    IN  A  192.168.1.200
jellyfin.yourdomain.com  IN  A  192.168.1.200
grafana.yourdomain.com   IN  A  192.168.1.200
```

Or use a wildcard DNS service like nip.io for testing:
- `http://jellyfin.192.168.1.200.nip.io`

## Security Considerations

- **IP exhaustion**: Limit services using LoadBalancer type
- **Network exposure**: All LoadBalancer services are externally accessible
- **Firewall**: Consider firewall rules to restrict access
- **Monitoring**: Watch for unauthorized services

## Best Practices

- Reserve generous IP range (at least 20 IPs)
- Document your IP allocations
- Use static IPs for critical services (annotations)
- Monitor IP pool usage
- Keep DHCP range separate
- Test failover behavior

## Resources

- MetalLB docs: https://metallb.universe.tf/
- Configuration guide: https://metallb.universe.tf/configuration/
- Concepts: https://metallb.universe.tf/concepts/
