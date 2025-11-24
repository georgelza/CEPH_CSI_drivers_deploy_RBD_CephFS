
## Deploy


### Cloning ceph-csi repo

```bash
git clone https://github.com/ceph/ceph-csi.git
cd ceph-csi
git checkout v3.15.0
```


## On CEPH cluster

We will first prepare our CEPH environment by creating our pools, volumes and file systems in addition to the user granted with the required permissions, before we start with the Kubernetes CSI deplpoyment itself.

Execute the following on your CEPH cluster

### Prepare the CephFS Pool's required

```bash
ceph osd pool create cephfs_data
ceph osd pool create cephfs_metadata
```

### Inspect

```bash
# Check the pool exists
ceph osd pool ls

# Check pool details
ceph osd pool get cephfs_data all
ceph osd pool get cephfs_metadata all
```

### Create the Filesystem onto the pools 

```bash
ceph fs new cephfs cephfs_metadata cephfs_data
```

### Use the returned "data pools:" value in storageclass for the "pools" value.

```bash
ceph fs ls
```
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]

```bash
ceph fs volume create cephfs
```

### Inspecting, List all the file system volumes

```bash
ceph fs volume ls
```

[
    {
        "name": "cephfs"
    }
]

### Get detailed info about a specific filesystem

```bash
ceph fs get cephfs
```

### Show filesystem status

```bash
ceph fs status cephfs
```

### Create a Subvolume Group

The subvolume group ensures that all CephFS volumes created by CSI are stored in one logical group

```bash
ceph fs subvolumegroup create cephfs csi
```

### Create a Cephfs User for for the secret that will be used via the storageclass

Cleanup, if we tried before...

```bash
ceph auth del client.k8s-rbd-user
ceph auth ls | grep k8s-rbd-user
```

```bash
ceph auth get-or-create client.k8s-cephfs-user \
  mds "allow rw fsname=cephfs path=csi" \
  mgr "allow rw" \
  mon "allow r fsname=cephfs" \
  osd "allow rwx tag cephfs data=cephfs" 

# or 

# Modify, we're over assigning permissions!
ceph auth caps client.k8s-cephfs-user \
  mds "allow rw fsname=cephfs path=csi" \
  mgr "allow rw" \
  mon "allow r fsname=cephfs" \
  osd "allow rwx tag cephfs data=cephfs" 
```

### Inspect

```bash
ceph auth get client.k8s-cephfs-user

[client.k8s-rbd-user]
	key = AQC7bh1pg1qRCRAAh2fL+ZfaOUtjyMd8q9IO5A==
	caps mds = "allow *"
	caps mgr = "allow *"
	caps mon = "allow *"
	caps osd = "allow * pool=k8s-pool"
```

### Ceph User Credentials

Lets get the "key", basically the random password created for the user created 

```bash
ceph auth get-key client.k8s-cephfs-user
```

### Lets create our manifest configuration files

Copy the above returned information into cephfs-secrets.yaml

```bash
cat <<EOF > cephfs-secrets.yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: csi-cephfs-secret
    namespace: ceph-csi-cephfs
  type: Opaque
  stringData:
    userID: k8s-cephfs-user
    userKey: AQASQB9pEcMbBBAAOjG0ond+gdQ8aJOAUHLBuw==
EOF
```

### Get Monitor IPs and Cluster FSID

```bash
ceph mon dump
ceph fsid
```

### Now, we need chart values file and add CEPH cluster ID and mon nodes list, from above

```bash
cat <<EOF > cephfs-values.yaml
  csiConfig: 
    - clusterID: "32a09c66-a6c3-4329-8339-15510d2ea9e0"
      monitors:
        - "172.16.10.51:6789"
        - "172.16.10.52:6789" 
        - "172.16.10.53:6789"
      cephFS:
        subvolumeGroup: "csi"
  provisioner:
    name: provisioner
    replicaCount: 2

  # Disable Helm-managed resources since you're creating them separately
  storageClass:
    create: false

  secret:
    create: false
EOF
```

### Lets compile our Storage Class definition

```bash
cat > cephfs-sc.yaml<< EOF
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: csi-cephfs-sc
  provisioner: cephfs.csi.ceph.com
  parameters:
    clusterID: 32a09c66-a6c3-4329-8339-15510d2ea9e0
    fsName: cephfs
    pool: cephfs_data
    mounter: kernel
    
    # Secret references
    csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
    csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi-cephfs
    csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
    csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi-cephfs
    csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
    csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi-cephfs

  reclaimPolicy: Delete
  allowVolumeExpansion: true
  volumeBindingMode: Immediate
EOF
```

## Now on the Kube cluster

Time to start deploying.

```bash
cd charts/ceph-csi-cephfs
```

```bash
cat <<EOF > cephfs-ns.yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: ceph-csi-cephfs
EOF
```

### Create namespace on kubernetes

```bash
kubectl apply -f cephfs-ns.yaml
```

### Let’s install Ceph CSI Cephfs chart on Kuberntes

```bash
helm install -n ceph-csi-cephfs ceph-csi-cephfs --values cephfs-values.yaml ./
```

### Check CSI Installation Status

```bash
helm status ceph-csi-cephfs -n ceph-csi-cephfs
kubectl rollout status deployment -n ceph-csi-cephfs
kubectl get pods -n ceph-csi-cephfs -o wide
```

### Create our CephFS Secret

```bash
kubectl apply -f cephfs-secrets.yaml
```

### Create CephFS StorageClass

```bash
kubectl apply -f cephfs-sc.yaml
```

### Verify Storageclass Build

```bash
kubectl get storageclass csi-cephfs-sc -o yaml | grep pool
```

### Create our test pvc and container

```bash
kubectl apply -f cephfs-pod-with-pvc.yaml
```

### Verify PVC and pod deployment

```bash
kubectl get pvc -n ceph-csi-cephfs

kubectl describe pvc/cephfs-pvc -n ceph-csi-cephfs

kubectl exec -n ceph-csi-cephfs pvc-demo -- cat /mnt/cephfs/data.txt
```


## Uninstall/Cleanup CephFS CSI stack

```bash
kubectl delete -f cephfs-pod-with-pvc.yaml
kubectl delete -f cephfs-sc.yaml
helm uninstall ceph-csi-cephfs -n ceph-csi-cephfs
kubectl delete -f cephfs-secrets.yaml
kubectl delete -f cephfs_ns.yaml
helm list -n ceph-csi-cephfs
```
