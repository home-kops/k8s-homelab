# Storage

This module provides persistent storage capabilities for your Kubernetes cluster.

## What's included?

### [NFS Provisioner](./nfs/README.md)
A dynamic storage provisioner that creates persistent volumes using an NFS server.

**What it does**: Automatically creates and manages storage volumes backed by NFS when applications request persistent storage.

## Why you need this

Kubernetes containers are ephemeral - when they restart, any data stored inside them is lost. Persistent storage solves this by:

- Storing data outside containers (on your NFS server)
- Surviving pod restarts and rescheduling
- Allowing data to be shared between multiple pods

## How it works

1. An application requests storage by creating a PersistentVolumeClaim (PVC)
2. The NFS provisioner sees the request
3. It creates a directory on your NFS server
4. It creates a PersistentVolume (PV) pointing to that directory
5. It binds the PV to the PVC
6. The application can now mount and use the storage

## Deployment

Storage must be deployed **first**, before any other services:

```bash
./storage/tooling/bootstrap
```

Other services (databases, media servers, monitoring tools) depend on persistent storage being available.

## Configuration

Configure your NFS server details in [nfs/values.yaml](./nfs/values.yaml):

```yaml
nfs:
  server: your-nfs-server-ip
  path: /path/to/shared/directory
```

## Storage Class

After deployment, a StorageClass named `nfs-sc` is available. Applications reference it in their PVCs:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  storageClassName: nfs-sc
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```
