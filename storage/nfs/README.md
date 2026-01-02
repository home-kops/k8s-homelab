# NFS Provisioner

Dynamic storage provisioner that automatically creates persistent volumes using an NFS server.

## What is the NFS Provisioner?

The NFS Provisioner automatically creates storage volumes for your applications. When an app says "I need 10GB of storage," the provisioner creates it on your NFS server without you having to manually set anything up.

## What it does

- **Dynamic provisioning**: Automatically creates storage when requested
- **NFS-backed**: Uses your existing NFS server for storage
- **ReadWriteMany**: Multiple pods can read/write to the same volume simultaneously
- **Automatic cleanup**: Deletes storage when no longer needed
- **StorageClass integration**: Works with standard Kubernetes storage APIs

## How it works

1. An application creates a PersistentVolumeClaim (PVC) requesting storage
2. The NFS provisioner sees the request
3. It creates a directory on your NFS server
4. It creates a PersistentVolume (PV) pointing to that directory
5. It binds the PV to the application's PVC
6. The application mounts the volume and uses it
7. When the PVC is deleted, the provisioner deletes the directory

## Why NFS?

NFS is ideal for homelabs because:
- **Shared storage**: Multiple pods can access the same data
- **Simple setup**: Easy to configure on most home servers
- **Existing hardware**: Use your NAS or file server
- **Centralized backup**: Back up one NFS server instead of many volumes

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
nfs:
  server: 192.168.1.100        # Your NFS server IP
  path: /mnt/storage/k8s       # Path on NFS server
  
storageClass:
  name: nfs-sc                 # Storage class name
  reclaimPolicy: Retain        # Keep data when PVC deleted
  archiveOnDelete: true        # Archive instead of immediate delete
```

## Prerequisites

Before deploying, ensure:
1. NFS server is running and accessible from cluster nodes
2. NFS exports are configured correctly
3. Cluster nodes have NFS client tools installed

### NFS Server Setup

On your NFS server (example for Linux):

```bash
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
