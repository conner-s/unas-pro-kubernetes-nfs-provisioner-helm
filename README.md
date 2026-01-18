# unas-pro-kubernetes-nfs-provisioner
A Kubernetes NFS provisioner compatible with the UNAS Pro and Hashicorp Vault.

## Prerequisites

Please make sure you have added every IP address of your cluster nodes to the
UNAS Pro. This can be done in the web UI of the drive under:

```text 
Settings > Services > Add NFS connections
```

You will also need the IP address of your UNAS Pro drive (found in the Unifi web UI topology)
and the path to your NFS share (e.g., `/var/nfs/shared/<folder>/<subfolder>/`).

## Installation

### Option 1: Using Helm (Recommended)

Install the NFS provisioner using Helm:

```shell
helm install nfs-provisioner ./charts/nfs-provisioner \
  --set nfs.server=<ip-address-of-UNAS-Pro> \
  --set nfs.path=/var/nfs/shared/<folder>/<subfolder>/
```

Or create a custom values file (`my-values.yaml`):

```yaml
nfs:
  server: "192.168.1.100"
  path: "/var/nfs/shared/kubernetes/data/"

storageClass:
  name: nfs-client
  defaultClass: true
```

Then install with:

```shell
helm install nfs-provisioner ./charts/nfs-provisioner -f my-values.yaml
```

To install in a specific namespace:

```shell
helm install nfs-provisioner ./charts/nfs-provisioner \
  --namespace storage \
  --create-namespace \
  --set nfs.server=<ip-address-of-UNAS-Pro> \
  --set nfs.path=/var/nfs/shared/<folder>/<subfolder>/
```

### Option 2: Using kubectl (Manual)

First add the IP address of your UNAS Pro drive to `deployment.yml`.
Also add the folder names of your shared drives and potentially subfolders.

Then apply Kubernetes files:

```shell
kubectl apply -f service-account.yml
kubectl apply -f cluster-role.yml
kubectl apply -f cluster-role-binding.yml
kubectl apply -f storage-class.yml
kubectl apply -f deployment.yml
```

After applying the Kubernetes resources, an NFS client provisioner will start running on
your cluster.

## Configuration Options

The following table lists the configurable parameters of the chart:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `nfs.server` | IP address of the UNAS Pro NFS server | `""` |
| `nfs.path` | Path to the NFS share | `""` |
| `image.repository` | Container image repository | `registry.k8s.io/sig-storage/nfs-subdir-external-provisioner` |
| `image.tag` | Container image tag | `v4.0.2` |
| `image.pullPolicy` | Container image pull policy | `IfNotPresent` |
| `provisionerName` | Provisioner name for the storage class | `nfs.csi.k8s.io` |
| `storageClass.name` | Name of the storage class | `nfs-client` |
| `storageClass.defaultClass` | Set as default storage class | `true` |
| `storageClass.archiveOnDelete` | Archive PVs on delete | `"true"` |
| `storageClass.reclaimPolicy` | Reclaim policy for PVs | `Retain` |
| `storageClass.mountOptions` | NFS mount options | `["nfsvers=3", "nolock"]` |
| `replicaCount` | Number of provisioner replicas | `1` |
| `resources` | CPU/Memory resource requests/limits | `{}` |
| `nodeSelector` | Node labels for pod scheduling | `{}` |
| `tolerations` | Pod tolerations | `[]` |
| `affinity` | Pod affinity rules | `{}` |

## Testing the Provisioner

Try it out by installing the HashiCorp Vault. 
Also see [HashiCorp documentation](https://developer.hashicorp.com/vault/docs/platform/k8s/helm)

```shell
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault
```

Now find the pod in your Kubernetes cluster in the namespace `default`.
Shell into it and use the command:

```vault operator init```

This will give you something like this:

```text
Unseal Key 1: <token>
Unseal Key 2: <token>
Unseal Key 3: <token>
Unseal Key 4: <token>
Unseal Key 5: <token>

Initial Root Token: <token>

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated root key. Without at least 3 keys to
reconstruct the root key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

Store the keys and have fun!

PS: Shout out to woutceu for mentioning the mount options.
https://community.ui.com/questions/NFS-File-shares-in-UNAS/fa03aa65-afec-4106-90bd-77c7b6e044c4#answer/bf4c757e-83c9-4870-8278-d006d1dbc19b
