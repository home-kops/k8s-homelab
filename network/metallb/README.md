# MetalLB

Load balancer providing external IP addresses to services in bare metal cluster.

## Deployment

**Manually bootstrapped** via [tooling/bootstrap](../tooling/bootstrap) script.

Deployed via **official Helm chart** ([kustomization.yaml](./kustomization.yaml)) with custom IP pool configuration in [ip-addresspool.yaml](./ip-addresspool.yaml).

## Configuration

### IP Address Pool

**Single IP** allocated ([ip-addresspool.yaml](./ip-addresspool.yaml)):
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
spec:
  addresses:
    - 192.168.10.99/32
```

Assigned to Traefik LoadBalancer service.

### Layer 2 Advertisement

**Control plane nodes only**:
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-advertisement
spec:
  ipAddressPools:
    - default-pool
  nodeSelectors:
    - matchLabels:
        node-role.kubernetes.io/control-plane: ""
```

Restricting to control plane ensures stable network connectivity for ingress traffic.

### How It Works

1. Traefik requests LoadBalancer IP
2. MetalLB assigns `192.168.10.99`
3. MetalLB on control plane responds to ARP requests
4. External clients reach Traefik at `192.168.10.99:80/443`

## Resources

Default Helm chart resource allocations.

## References

- [MetalLB Documentation](https://metallb.universe.tf/)
- [Helm Chart](https://github.com/metallb/metallb/tree/main/charts/metallb)### L2Advertisement

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
