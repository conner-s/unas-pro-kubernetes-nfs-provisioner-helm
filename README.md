# UNAS Pro NFS Configuration for Kubernetes

Guide for configuring the official [Kubernetes CSI NFS Driver](https://github.com/kubernetes-csi/csi-driver-nfs) to work with the UniFi UNAS Pro network storage device.

> **Note:** This repository previously contained a custom Helm chart, but we now recommend using the official CSI driver which is actively maintained by Kubernetes SIG-Storage and provides additional features like volume snapshots, cloning, and proper volume expansion.

## Prerequisites

### 1. Configure UNAS Pro NFS Access

Add the IP addresses of all your Kubernetes cluster nodes to the UNAS Pro. This can be done in the UNAS Pro web UI:

```
Settings > Services > Add NFS connections
```

### 2. Gather Required Information

You will need:
- **UNAS Pro IP address**: Found in the UniFi Network topology view
- **NFS share path**: e.g., `/var/nfs/shared/<folder>/<subfolder>/`

## Installation

### Step 1: Install the Official CSI NFS Driver

```shell
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --set driver.mountPermissions=0
```

### Step 2: Create a StorageClass for UNAS Pro

Create a file named `storageclass-unas-pro.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-unas-pro
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs.csi.k8s.io
parameters:
  server: <UNAS-PRO-IP-ADDRESS>
  share: /var/nfs/shared/<folder>/<subfolder>/
  # subDir: ""  # Optional: subdirectory under the share
  # onDelete: retain  # Options: delete, retain, archive
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - nfsvers=3   # Required for UNAS Pro compatibility
  - nolock      # Required for UNAS Pro compatibility
```

Apply the StorageClass:

```shell
kubectl apply -f storageclass-unas-pro.yaml
```

## UNAS Pro-Specific Mount Options

The UNAS Pro requires specific NFS mount options to function correctly:

| Option | Description |
|--------|-------------|
| `nfsvers=3` | Use NFS version 3 protocol (required for UNAS Pro) |
| `nolock` | Disable file locking (required for UNAS Pro) |

These options are included in the StorageClass example above.

## Example: Creating a PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-unas-pro
  resources:
    requests:
      storage: 10Gi
```

## Example: Using the PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: nfs-storage
          mountPath: /data
  volumes:
    - name: nfs-storage
      persistentVolumeClaim:
        claimName: my-nfs-pvc
```

## Testing with HashiCorp Vault

A good way to test your NFS provisioner is by installing HashiCorp Vault:

```shell
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --set server.dataStorage.storageClass=nfs-unas-pro
```

After installation, initialize Vault:

```shell
kubectl exec -it vault-0 -- vault operator init
```

## Advanced Configuration

### Static Provisioning (Pre-existing NFS Share)

For using a specific existing directory on the NFS server:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-unas-pro
  csi:
    driver: nfs.csi.k8s.io
    volumeHandle: unique-volume-id
    volumeAttributes:
      server: <UNAS-PRO-IP-ADDRESS>
      share: /var/nfs/shared/<folder>/<subfolder>/
  mountOptions:
    - nfsvers=3
    - nolock
```

### Volume Snapshots

The official CSI driver supports volume snapshots. First, enable the snapshot controller:

```shell
helm upgrade csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --set externalSnapshotter.enabled=true
```

Create a VolumeSnapshotClass:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: nfs-snapclass
driver: nfs.csi.k8s.io
deletionPolicy: Delete
```

## Why Use the Official CSI Driver?

| Feature | Official csi-driver-nfs |
|---------|------------------------|
| Volume Snapshots | ✅ Supported |
| Volume Cloning | ✅ Supported |
| Volume Expansion | ✅ Full CSI support |
| fsGroupPolicy | ✅ Supported |
| Maintenance | Active (Kubernetes SIG-Storage) |
| UNAS Pro Compatibility | ✅ Via mount options |

## Troubleshooting

### Mount fails with "access denied"
- Verify the UNAS Pro has your node IPs added to NFS connections
- Check firewall rules allow NFS traffic (ports 111, 2049)

### "Stale file handle" errors
- Ensure `nolock` mount option is present
- Verify the NFS path exists on the UNAS Pro

### Permission issues
- Check the `mountPermissions` parameter in the CSI driver
- Verify the NFS export permissions on the UNAS Pro

## References

- [Official CSI NFS Driver](https://github.com/kubernetes-csi/csi-driver-nfs)
- [CSI Driver Parameters](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/driver-parameters.md)
- [UNAS Pro NFS Community Discussion](https://community.ui.com/questions/NFS-File-shares-in-UNAS/fa03aa65-afec-4106-90bd-77c7b6e044c4#answer/bf4c757e-83c9-4870-8278-d006d1dbc19b)
