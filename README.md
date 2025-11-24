

## Deploying CEPH CSI drivers (both RBD and CephFS) onto a Kubernetes cluster into separate namespaces

(25 Nov2025)

**Overview**

Deploying, specifically both the RBD and CephFS CSI stacks onto a Kubernetes cluster into separate namespaces

**Well firstly, what is CEPH, you may ask.**

*Ceph is an open-source, software-defined storage platform that provides unified object, block, and file storage within a single, scalable cluster. It links multiple storage servers into a single distributed system, making it fault-tolerant and highly available without a single point of failure. Ceph is known for its flexibility, ability to run on commodity hardware, and compatibility with other open-source projects like OpenStack and Kubernetes.* 

See [Ceph: An Overview](https://fabreur.medium.com/ceph-an-overview-e971c00ded93)


This all started with my desire to deploy various containers onto my [Kubespray](https://github.com/kubernetes-sigs/kubespray) cluster.


- [PostgreSQL](https://www.postgresql.org)
- [MinIO](https://www.min.io)
- [Neo4J](https://neo4j.com)
- [Apache Kafka using the Confluent](https://confluent.io) deployment stack
- [Apache Flink](https://flink.apache.org)
- [Apache Fluss (Incubating)](https://fluss.apache.org)
- [Apache Iceberg](https://iceberg.apache.org)
- [Apache Paimon](https://paimon.apache.org)

For which some require a S3 based object storage, some requiring File system based PVC storage and some just happy with a blob of PVC.

For the above I basically required whats called a CSI driver/stack, CSI meaning Container Storage Interface allowing my Kubernetes cluster to use external persistent storage.

As my environment is all hosted as VM's on a [Proxmox](https://www.proxmox.com/en/) cluster and it iself includes [CEPH](https://ceph.io/en/)... well that then lead to the below.


## About my Environment

Below is a little overview of my lab.

3 x minicomputers 

- i5-1335U  [12 x 13th Gen Intel(R) Core(TM) i5-1335U (1 Socket)]
- 32GB RAM, 
- 4 TB NVMe card, partitioned into 60GB for [Proxmox](https://www.proxmox.com/en/) and balance, as per below

- pmox1     
  - 172.16.10.51 (bridge0, 2 x 2.5GbE) 
  - 172.16.40.51 (bridge1, 2 x 10GbE SFP+)
- pmox2     
  - 172.16.10.52 
  - 172.16.40.51
- pmox3     
  - 172.16.10.53
  - 172.16.40.53
  
The base Hypervisor is [Proxmox 9.0.11](https://www.proxmox.com/en/), configured into a HA Cluster.

I then configured a CEPH cluster using the Proxmox included CEPH capability. The CEPH cluster is configured with OSD's (think disks) using the remaining space from the NVME SSD's.

On this CEPH storage I then provisioned pools, (imgpool was created via the [Proxmox](https://www.proxmox.com/en/) CEPH GUI interface, the others via the below `*_deploy.md` documented steps).

- imgpool is used as the storage pool hosting the below 6 VM's images.
- k8s_pool for RBD - see rmd_deploy.md
- k8s_data - see cephfs_deploy.md
- k8s_metadata - see cephfs_deploy.md

To deploy me entire environment (and I rediscovered, due to my own fault, I by accident deleted my pool that host my VM images, using a master clone and Ansible is definitely advantages). 

Script as much as you can using [Ansible](https://docs.ansible.com), make notes in README.md 's files located your code directories, you will thank me later.

I build up a master image ([Ubuntu](https://ubuntu.com) 24.10), this is hosted/presented via an NFS mounted volume which is hosted on my [TrueNAS](https://www.truenas.com).

The master image was pre configured with some users (`ansible`) and included the `ssh's key's` from my MBP, before a template was created.

This template image was then cloned into the following 6 VM's.

- 301 k8s-ubuntu-1  pmox1   master/control plane
  - 172.16.10.71    (net0, Public Access, over bridge0)
  - 172.16.30.71    (net1, K8S inter node comm, over bridge1)
  - 172.16.40.71    (net2, IO comms, traffic between nodes and between nodes and the TrueNAS, over bridge1)

- 302 k8s-ubuntu-2  pmox2   master/control plane
  - 172.16.10.72
  - 172.16.30.72
  - 172.16.40.72

- 303 k8s-ubuntu-3  pmox3   master/control plane
  - 172.16.10.73
  - 172.16.30.73
  - 172.16.40.73

- 304 k8s-ubuntu-4  pmox1   worker
  - 172.16.10.74
  - 172.16.30.74
  - 172.16.40.74

- 305 k8s-ubuntu-5  pmox2   worker
  - 172.16.10.75
  - 172.16.30.75
  - 172.16.40.75

- 306 k8s-ubuntu-6  pmox3   worker
  - 172.16.10.76
  - 172.16.30.76
  - 172.16.40.76
  
NOTE: for the Kubernetes and Ceph and [MetalLB](https://metallb.io) stack to work the above VM's need to be configured with processor as `x86-63-v2`

Each VM is configured with:

- 2 vCPU 
- 4 GB RAM
- 20 GB disk

Once the 6 x VM's were up and running I then used KubeSpray to deploy my Kubernetes cluster.

See: [Setting up a Kubernetes cluster with Kubespray]([https://medium.com/@leonardo.bueno/setting-up-a-kubernetes-cluster-with-kubespray-1bf4ce8ccd73)

At this point, we are now ready to deploy our Ceph CSI stack onto our Kubernetes cluster that will act as "client interface" allowig us use storage as available from the Proxmox hosted Ceph cluster. Allot of the "other" examples out on the interwebs want to show you how to deploy a CEPH cluster onto your Kubernetes cluster. I opted to rather "plug" into the one I already had.

Once you clone my project GIT repo, you will find 2 folders. You will see their names match up to the CEPH CSI GIT repo structure... read on you will see why.

So between the 2 folders we have everything required to deploy the Ceph RBD and Ceph FS CSI drivers.

I've gone with a dedicated namespace for each. Quickly figured it out was not simple, deploying just one CSI stack, is easy, but as soon as you try both the RBD and cephfs, you run into some fun, in the end I think the management would be simpler though.

The biggest thing, ye accurate copy/past'ing is important, hehehe and then the 2 directories each contain a `values.yaml` file, as included in the Ceph git repo also. The biggest change in these files are the modification of the `container` and `service` ports numbers.

The trick in the end was to modify the container and service ports for the RBD CSI install onto ports 8082 and for Cephfs CSI install onto 8081.

I've decided to stick with the "suggested" namespaces, storage class and secret names:

**Namespaces:**

- ceph-csi-cephfs
- ceph-csi-rbd

**Storageclass:**

- csi-cephfs-sc
- csi-rbd-sc

**Secrets:**
The secrets I did decide to relocate into the dedicate respective namespace though, the base script normally deploys the secrets into the `default` namespace.

- csi-cephfs-secret
- csi-rbd-secret


### Deploying.

To deploy the stack, copy the contants of `ceph-csi-cephfs` into the `ceph-csi/charts/ceph-csi-cephfs` directory as located in the GIT repo cloned and similarly copy the contents of `ceph-csi-rbd` into the `ceph-csi/ceph-csi/rbd` directory.

Now follow the `rbd-deploy.md` to deploy the RBD CSI stack and the `cephfs-deploy.md` to deploy the Cephfs CSI stack.

Once You done the above you will end with something that should look like this.

```
kubectl get sc
NAME                   PROVISIONER           RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
csi-cephfs-sc          cephfs.csi.ceph.com   Delete          Immediate           true                   16h
csi-rbd-sc (default)   rbd.csi.ceph.com      Delete          Immediate           true                   83s
kubectl get storageclass csi-rbd-sc -n ceph-csi-rbd
NAME                   PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
csi-rbd-sc (default)   rbd.csi.ceph.com   Delete          Immediate           true                   3m40s

kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-724518f3-3207-435c-93ed-dd4a0a9c74ac   1Gi        RWO            Delete           Bound    ceph-csi-rbd/rbd-pvc         csi-rbd-sc      <unset>                          3m35s
pvc-b08b5aaf-8b51-4f49-99d1-41f0ab9bc812   1Gi        RWX            Delete           Bound    ceph-csi-cephfs/cephfs-pvc   csi-cephfs-sc   <unset>                          16h
```


## Credits

While trying to figure out how to get the above done, and allot of searching, I could not find "exactly/perfectly" what I wanted.

But, then I got lucky and found [Ceph Storage Integration with Kubernetes using Ceph CSI](https://medium.com/@satishdotpatel/ceph-integration-with-kubernetes-using-ceph-csi-c434b41abd9c) by [Satish Patel](https://medium.com/@satishdotpatel).

This got me going with the RBD CSI stack. using this pattern and allot of trying, googling, claude'ing etc I got the Cephfs figured out.

