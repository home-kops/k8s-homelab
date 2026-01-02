# Jellyfin

Media server for organizing and streaming personal media libraries.

## Configuration

### Container Image

**LinuxServer.io Jellyfin image**: [ghcr.io/linuxserver/jellyfin](https://github.com/linuxserver/docker-jellyfin)

### Storage

**Dual storage strategy:**

1. **Configuration PVC** (10Gi on `nfs-sc`):
   - Stores Jellyfin metadata, settings, library database
   - Persistent across pod restarts

2. **NFS Media Mount**:
   ```yaml
   nfs: <path:secret/data/infra#nfs>
   ```
   - Direct mount to shared media library
   - NFS server IP resolved from Vault
   - Read-only access to prevent accidental modifications

### Automated Backups

CronJob backs up configuration daily:
```yaml
schedule: "0 15 * * *"  # 3 PM daily
```
Retention: 3 days

## Resources

- **CPU**: Not explicitly limited
- **Memory**: Not explicitly limited
- **Storage**: 10Gi PVC + NFS mount
- **Node Selector**: `size: m` (medium nodes)

## Access

**IngressRoute**: `https://jellyfin.{{ .Values.domain }}`

Domain resolved from Vault: `<path:secret/data/infra#domain>`

## References

- [Jellyfin Documentation](https://jellyfin.org/docs/)
- [LinuxServer.io Jellyfin Image](https://docs.linuxserver.io/images/docker-jellyfin)
```
https://jellyfin.<your-domain>
```

The IngressRoute is defined in [templates/ingressroute.yaml](./templates/ingressroute.yaml) with automatic TLS.

## Resource Scheduling

Deployed to medium-sized nodes:
```yaml
nodeSelector:
  size: m
```

Jellyfin needs sufficient CPU/RAM for:
- Metadata processing
- Library scanning
- Potential transcoding (if hardware acceleration unavailable)

## Deployment

Managed by ArgoCD. Changes to values.yaml are automatically synced to the cluster.

## Monitoring

Check Jellyfin status:
```bash
kubectl get pods -n media -l app=jellyfin
kubectl logs -n media -l app=jellyfin
kubectl get pvc -n media
```

Check backup CronJob:
```bash
kubectl get cronjob -n media
kubectl get jobs -n media | grep backup
```

## Troubleshooting

**Pod not starting:**
- Check PVC is bound: `kubectl get pvc -n media`
- Verify NFS server is accessible
- Check node selector matches available nodes

**Can't access media:**
- Verify NFS mount is working
- Check file permissions on NFS share
- Ensure PUID/PGID match NFS file ownership

**Slow performance:**
- Transcoding without hardware acceleration is CPU-intensive
- Check resource limits
- Consider enabling GPU passthrough for transcoding
