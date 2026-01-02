# NFS Provisioner

Dynamic storage provisioner creating persistent volumes on NFS server.

## Deployment

**Manually bootstrapped** via [tooling/bootstrap](../tooling/bootstrap) script.

Deployed via **local Helm chart** with templates in [templates/](./templates/):
- [deployment.yaml](./templates/deployment.yaml)
- [serviceaccount.yaml](./templates/serviceaccount.yaml)
- [storageclass.yaml](./templates/storageclass.yaml)

## Configuration

### NFS Server

Server details from Vault:
```yaml
nfs:
  server: <path:secret/data/infra#k8s_nfs>
  path: /k8s-homelab
```

### Storage Class

Creates default StorageClass `nfs-sc`:
```yaml
storageClass:
  name: nfs-sc
  defaultClass: true
  reclaimPolicy: Retain
  archiveOnDelete: false
```

- **defaultClass: true**: Used when PVCs omit `storageClassName`
- **reclaimPolicy: Retain**: PVs persist after PVC deletion
- **archiveOnDelete: false**: Immediate directory deletion

### Container Image

[registry.k8s.io/sig-storage/nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

### Control Plane Deployment

**Runs on control plane nodes only**:
```yaml
nodeSelector:
  node-role.kubernetes.io/control-plane: ""

tolerations:
  - key: node-role.kubernetes.io/control-plane
    effect: NoSchedule
```

Ensures provisioner high availability and reduces network hops to storage.

## Resources

Default container resource allocations (not customized).

## References

- [NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [Kubernetes Storage Documentation](https://kubernetes.io/docs/concepts/storage/)
# Install NFS server
sudo apt install nfs-kernel-server

# Create export directory
sudo mkdir -p /mnt/storage/k8s
sudo chown nobody:nogroup /mnt/storage/k8s
sudo chmod 777 /mnt/storage/k8s

# Configure export
echo "/mnt/storage/k8s *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Apply changes
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

### Cluster Node Setup

On each Kubernetes node:

```bash
# Install NFS client
sudo apt install nfs-common
```

## Deployment

Deploy via the storage bootstrap script (must be first):

```bash
./storage/tooling/bootstrap
```

## Using Storage

Applications request storage using PersistentVolumeClaims:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-storage
spec:
  storageClassName: nfs-sc
  accessModes:
    - ReadWriteMany    # Multiple pods can read/write
  resources:
    requests:
      storage: 10Gi
```

Then mount it in a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: my-image
      volumeMounts:
        - name: storage
          mountPath: /data
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: my-app-storage
```

## Access Modes

- **ReadWriteOnce (RWO)**: One pod can mount as read-write
- **ReadOnlyMany (ROX)**: Many pods can mount as read-only
- **ReadWriteMany (RWX)**: Many pods can mount as read-write (NFS specialty!)

## Reclaim Policies

- **Retain**: Keep data when PVC deleted (manual cleanup required)
- **Delete**: Automatically delete data when PVC deleted
- **Recycle**: Deprecated

For production data, use **Retain** and manage cleanup manually.

## Troubleshooting

**Pods stuck in "ContainerCreating"**: Check NFS connectivity, verify exports

**Permission denied**: Check NFS export options, ensure `no_root_squash` if needed

**Mount failures**: Verify NFS client tools installed on nodes

**Slow performance**: NFS can be slower than local storage, consider for media/backups but not databases

## Monitoring

Check PVCs and PVs:
```bash
kubectl get pvc -A
kubectl get pv
```

Check provisioner logs:
```bash
kubectl logs -n storage deployment/nfs-client-provisioner
```
