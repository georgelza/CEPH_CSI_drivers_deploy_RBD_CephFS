

## Deploying CEPH CSI drivers (both RBD and CephFS) onto a Kubernetes cluster into separate namespaces

(25 Nov2025)

**Overview**

Deploying, specifically both the RBD and CephFS CSI stacks onto a Kubernetes cluster into separate namespaces using [HELM](https://helm.sh) and YAML configuration files.


**Well firstly, what is CEPH, you may ask.**

*Ceph is an open-source, software-defined storage platform that provides unified object, block, and file storage within a single, scalable cluster. It links multiple storage servers into a single distributed system, making it fault-tolerant and highly available without a single point of failure. Ceph is known for its flexibility, ability to run on commodity hardware, and compatibility with other open-source projects like OpenStack and Kubernetes.* 

`<Diagram>`

See [Ceph: An Overview](https://fabreur.medium.com/ceph-an-overview-e971c00ded93)


**What is a CEPH Cluster**

A Ceph cluster is a distributed, scalable storage system that provides object, block, and file storage from a single, unified cluster. It uses a collection of daemons, including Ceph Monitors and Ceph OSD (Object Storage Daemons), to store and replicate data across multiple nodes running on commodity hardware. A key feature is the use of the CRUSH algorithm to automatically distribute data without a central bottleneck, ensuring reliability and performance. 

`<Diagram>`

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

For the above I basically required whats called a CSI driver/stack, CSI meaning Container Storage Interface allowing my Kubernetes cluster to use external storage for K8S persistent storage.

As my environment is all hosted as VM's on a [Proxmox](https://www.proxmox.com/en/) cluster and it iself includes [CEPH](https://ceph.io/en/)... well that then lead to the below.

[GIT REPO](https://github.com/georgelza/CEPH_CSI_drivers_deploy_RBD_CephFS)


## About my Environment

Below is a **little** overview of my lab.

<Diagram>

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

- 301 k8s-ubuntu-1  hosted on pmox1 as a master
  - 172.16.10.71    (net0, Public Access, over bridge0)
  - 172.16.30.71    (net1, K8S inter node comm, over bridge1)
  - 172.16.40.71    (net2, IO comms, traffic between nodes and between nodes and the TrueNAS, over bridge1)

- 302 k8s-ubuntu-2  hosted on pmox2 as a master
  - 172.16.10.72
  - 172.16.30.72
  - 172.16.40.72

- 303 k8s-ubuntu-3  hosted on pmox3 as a master
  - 172.16.10.73
  - 172.16.30.73
  - 172.16.40.73

- 304 k8s-ubuntu-4  hosted on pmox1 as a worker
  - 172.16.10.74
  - 172.16.30.74
  - 172.16.40.74

- 305 k8s-ubuntu-5  hosted on pmox2 as a worker
  - 172.16.10.75
  - 172.16.30.75
  - 172.16.40.75

- 306 k8s-ubuntu-6  hosted on pmox3 as a worker
  - 172.16.10.76
  - 172.16.30.76
  - 172.16.40.76


### /etc/hosts

```  
172.16.10.51    pmox1.<domain>.com            pmox1
172.16.30.51    pmox1-vm.<domain>.com         pmox1-vm
172.16.40.51    pmox1-io.<domain>.com         pmox1-io

172.16.10.52    pmox2.<domain>.com            pmox2
172.16.30.52    pmox2-vm.<domain>.com         pmox2-vm
172.16.40.52    pmox2-io.<domain>.com         pmox2-io

172.16.10.53    pmox3.<domain>.com            pmox3
172.16.30.53    pmox3-vm.<domain>.com         pmox3-vm
172.16.40.53    pmox3-io.<domain>.com         pmox3-io

172.16.10.24    vaultx.<domain>.com           vaultx
172.16.40.24    vaultx-io.<domain>.com        vaultx-io

# Kubespray Prd Cluster
172.16.10.61    ubuntu-1.<domain>.com         ubuntu-1
172.16.30.61    ubuntu-1-vm.<domain>.com      ubuntu-1-vm
172.16.40.61    ubuntu-1-io.<domain>.com      ubuntu-1-io

172.16.10.62    ubuntu-2.<domain>.com         ubuntu-2
172.16.30.62    ubuntu-2-vm.<domain>.com      ubuntu-2-vm
172.16.40.62    ubuntu-2-io.<domain>.com      ubuntu-2-io

172.16.10.71    k8s-ubuntu-1.<domain>.com     k8s-ubuntu-1
172.16.30.71    k8s-ubuntu-1-vm.<domain>.com  k8s-ubuntu-1-vm
172.16.40.71    k8s-ubuntu-1-io.<domain>.com  k8s-ubuntu-1-io

172.16.10.72    k8s-ubuntu-2.<domain>.com     k8s-ubuntu-2
172.16.30.72    k8s-ubuntu-2-vm.<domain>.com  k8s-ubuntu-2-vm
172.16.40.72    k8s-ubuntu-2-io.<domain>.com  k8s-ubuntu-2-io

172.16.10.73    k8s-ubuntu-3.<domain>.com     k8s-ubuntu-3
172.16.30.73    k8s-ubuntu-3-vm.<domain>.com  k8s-ubuntu-3-vm
172.16.40.73    k8s-ubuntu-3-io.<domain>.com  k8s-ubuntu-3-io

172.16.10.74    k8s-ubuntu-4.<domain>.com     k8s-ubuntu-4
172.16.30.74    k8s-ubuntu-4-vm.<domain>.com  k8s-ubuntu-4-vm
172.16.40.74    k8s-ubuntu-4-io.<domain>.com  k8s-ubuntu-4-io

172.16.10.75    k8s-ubuntu-5.<domain>.com     k8s-ubuntu-5
172.16.30.75    k8s-ubuntu-5-vm.<domain>.com  k8s-ubuntu-5-vm
172.16.40.75    k8s-ubuntu-5-io.<domain>.com  k8s-ubuntu-5-io

172.16.10.76    k8s-ubuntu-6.<domain>.com     k8s-ubuntu-6
172.16.30.76    k8s-ubuntu-6-vm.<domain>.com  k8s-ubuntu-6-vm
172.16.40.76    k8s-ubuntu-6-io.<domain>.com  k8s-ubuntu-6-io
```

**NOTE**: for the Kubernetes, Ceph and [MetalLB](https://metallb.io) stack to work the above VM's need to be configured with processor as `x86-63-v2`

Each VM is configured with:

- 2 vCPU 
- 4 GB RAM
- 20 GB disk

Once the 6 x VM's were up and running I then used KubeSpray to deploy my Kubernetes cluster.

See: [Setting up a Kubernetes cluster with Kubespray]([https://medium.com/@leonardo.bueno/setting-up-a-kubernetes-cluster-with-kubespray-1bf4ce8ccd73)

At this point, we are now ready to deploy our Ceph CSI stack onto our Kubernetes cluster that will act as "client interface" allowig us use storage as available from the Proxmox hosted Ceph cluster. Allot of the "other" examples out on the interwebs want to show you how to deploy a CEPH cluster onto your Kubernetes cluster. I opted to rather "plug" into the one I already had.

### The Deployment

Can be seen as the executing the following phases:

- Source CEPH Cluster information and Prepare Storage and User Access &Credentials
- Consolidate information from above into Yaml based configuration files
- Modify `values.yaml`
- Deploy Ceph CSI stack using `helm install -n <namespace> <app> --values <rbd or cephfs>-values.yaml`
- Deploy our configuration Yaml files kubectl apply -f <filename>, creating `csi-<rbd or cephfs>-secret` & `csi-<rbd or cephfs>-sc`
- Test/Validate by creating test PVC and pod.


Once you clone the [project GIT repo](https://github.com/georgelza/CEPH_CSI_drivers_deploy_RBD_CephFS), you will find 2 folders. You will see their names match up to the CEPH CSI GIT repo structure... read on you will see why.

So between the 2 folders we have everything required to deploy the Ceph RBD and Ceph FS CSI drivers.

I've gone with a dedicated namespace for each. 

I quickly figured out, it was not simple. Deploying just one CSI stack, is easy, but as soon as you try both the RBD and Cephfs, you run into some fun, in the end I think the usage of 2namespaces will make management simpler though.

The biggest thing, ye accurate copy/past'ing is important, hehehe and then the 2 directories each contain a `values.yaml` file, as included in the Ceph git repo also. The biggest change in these files are the modification of the `container` and `service` ports numbers.

The trick in the end was to modify both the `container` and `service` ports:

- Cephfs CSI install onto ports 8081
- RBD CSI install onto ports 8082 

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


To deploy the stack, copy the contants of `ceph-csi-cephfs` into the `ceph-csi/charts/ceph-csi-cephfs` directory as located in the GIT repo cloned and similarly copy the contents of `ceph-csi-rbd` into the `ceph-csi/charts/ceph-csi-rbd` directory.

Now follow the slightly more detail, with examples instructions in:

- `rbd-deploy.md` to deploy the RBD CSI stack.
- `cephfs-deploy.md` to deploy the Cephfs CSI stack.

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


## SUMMARY

Well, hope I got it all accurately documented and copied, Hope it’s of benefit for someone. 

Thanks for following. Till next time.

And like that we're done with our little trip down another Rabbit Hole.

## Credits

While trying to figure out how to get the above done, and allot of searching, I could not find "exactly/perfectly" what I wanted.

But, then I got lucky and found [Ceph Storage Integration with Kubernetes using Ceph CSI](https://medium.com/@satishdotpatel/ceph-integration-with-kubernetes-using-ceph-csi-c434b41abd9c) by [Satish Patel](https://medium.com/@satishdotpatel).

His Medium article got me going with the RBD CSI stack, using this pattern and allot of trying, googling, claude'ing etc I got the Cephfs figured out.


## About Me

I’m a techie, a technologist, always curious, love data, have for as long as I can remember always worked with data in one form or the other, Database admin, Database product lead, data platforms architect, infrastructure architect hosting databases, backing it up, optimizing performance, accessing it.  Data data data… it makes the world go round.

In recent years, pivoted into a more generic Technology Architect role, capable of full stack architecture.

George Leonard
georgelza@gmail.com
https://medium.com/@georgelza
