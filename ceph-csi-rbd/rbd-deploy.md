
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
# Create the Ceph Pool
ceph osd pool create k8s_pool 16 16
# Initialize the pool for RBD use
ceph osd pool application enable k8s_pool rbd
# Initialize RBD
rbd pool init k8s_pool
```

### Inspect

```bash
# Check the pool exists
ceph osd pool ls

# Check pool details
ceph osd pool get k8s_pool all
```

### Create a RBD User for for the secret that will be used via the storageclass

Cleanup, if we tried before...

```bash
ceph auth del client.k8s-rbd-user
ceph auth ls | grep k8s-rbd-user
```

```bash
ceph auth get-or-create-key client.k8s-user \
  mds 'allow *' \
  mgr 'allow *' \
  mon 'allow *' \
  osd 'allow * pool=k8s-pool' | tr -d '\n' | base64

# or 

# Modify, we're over assigning permissions!
ceph auth caps client.k8s-rbd-user \
  mds 'allow *' \
  mgr 'allow *' \
  mon 'allow *' \
  osd 'allow * pool=k8s_pool'
```

### Inspect

```bash
ceph auth get client.k8s-rbd-user

[client.k8s-rbd-user]
	key = AQC7bh1pg1qRCRAAh2fL+ZfaOUtjyMd8q9IO5A==
	caps mds = "allow *"
	caps mgr = "allow *"
	caps mon = "allow *"
	caps osd = "allow * pool=k8s_pool"
```

### Ceph User Credentials

Lets get the "key", basically the random password created for the user created, we will also base64 encode it.

### Encode key into base64

```bash
echo -n "AQC7bh1pg1qRCRAAh2fL+ZfaOUtjyMd8q9IO5A==" | base64 -w 0
QVFDN2JoMXBnMXFSQ1JBQWgyZkwrWmZhT1V0anlNZDhxOUlPNUE9PQ==
```

Now for the username

### base64 Encode the username

```bash
echo "k8s-rbd-user" | tr -d '\n' | base64
```

azhzLXJiZC11c2Vy

### Lets create our manifest configuration files

Copy the base64 encoded values from above into rbd-secret.yaml 

```bash
cat <<EOF >  rbd-secret.yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: csi-rbd-secret
    namespace: ceph-csi-rbd
  type: kubernetes.io/rbd
  data:
    userID: azhzLXJiZC11c2Vy
    userKey: QVFDN2JoMXBnMXFSQ1JBQWgyZkwrWmZhT1V0anlNZDhxOUlPNUE9PQ==
EOF
```

### Get Monitor IPs and Cluster FSID

```bash
ceph mon dump
ceph fsid
```

### Now, we need chart values file and add CEPH cluster ID and mon nodes list, from above

```bash
cat <<EOF > rbd-values.yaml
  csiConfig: 
    - clusterID: "32a09c66-a6c3-4329-8339-15510d2ea9e0"
      monitors:
        - "172.16.10.51:6789"
        - "172.16.10.52:6789" 
        - "172.16.10.53:6789"
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
cat <<EOF > rbd-sc.yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: csi-rbd-sc
    namespace: ceph-csi-rbd
    annotations:
      storageclass.kubernetes.io/is-default-class: "true"
  provisioner: rbd.csi.ceph.com
  parameters:
    clusterID: "32a09c66-a6c3-4329-8339-15510d2ea9e0"
    pool: k8s_pool
    imageFeatures: layering
    
    csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
    csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi-rbd
    csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
    csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi-rbd
    csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
    csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi-rbd
  reclaimPolicy: Delete
  allowVolumeExpansion: true
  mountOptions:
    - discard
EOF
```

## Now on the Kube cluster

Time to start deploying.

```bash
cd charts/ceph-csi-rbd
```

```bash
cat <<EOF > rbd-ns.yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: ceph-csi-rbd
EOF
```

### Create namespace on kubernetes

```bash
kubectl apply -f rbd-ns.yaml
```

### Let’s install the Ceph CSI RBD chart on Kuberntes

```bash
helm install -n ceph-csi-rbd ceph-csi-rbd --values rbd-values.yaml ./
```

### Check CSI Installation Status

```bash
helm status ceph-csi-rbd -n ceph-csi-rbd
kubectl rollout status deployment -n ceph-csi-rbd
kubectl get pods -n ceph-csi-rbd -o wide
```

### Create our RBD Secret

```bash
kubectl apply -f rbd-secret.yaml
```

### Create RBD StorageClass

```bash
kubectl apply -f rbd-sc.yaml
```

### Verify Storageclass Build

```bash
kubectl get sc
```

### Create our test pvc and container

```bash
kubectl apply -f rbd-pod-with-pvc.yaml
```

```bash
kubectl get pvc -n ceph-csi-rbd

kubectl describe pvc/rbd-pvc -n ceph-csi-rbd

kubectl exec -n ceph-csi-rbd pod/pvc-demo -- cat /mnt/rbd/data.txt
```


## Uninstall/Cleanup RBD CSI stack

```bash
kubectl delete -f rbd-pod-with-pvc.yaml
kubectl delete -f rbd-sc.yaml
kubectl delete -f rbd-secret.yaml
helm uninstall ceph-csi-rbd -n ceph-csi-rbd
kubectl delete -f rbd_ns.yaml
helm list -n ceph-csi-rbd
```