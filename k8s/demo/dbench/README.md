This folder contains sample YAMLs for running bench marking tests
using fio scripts present at [logdna/dbench](https://github.com/logdna/dbench/). 

To get optimal performance, it is essential to tune the storage settings for the
type of workload and as well as customize the storage to run along side 
application workloads.

Depending on the storage engine of choice, the tunables will vary. `Jiva` being
the most simple one to use, I will start with that and then follow up with `cStor`.

## Running bench marking tests using `Jiva`

Jiva storage engine involves - a target pod that receives IO from the application 
and then makes copies of data sends them synchronously to one or more replica pods
for persisting the data. The replica pods will write the data into host-path on 
the node where they are running. The host-path is configured by a CR called 
StoragePool, with default path as `/var/openebs`. 

Some of the ways to tune the `jiva` storage engine for benchmark tests.

### Configure the `Jiva` storage pool host-path 

Edit the default StoragePool CR and modify the host-path to point to the 
correct storage location. 

For instance, if I am using local ssds on GKE, the StoragePool would look like:

```
apiVersion: openebs.io/v1alpha1
kind: StoragePool
metadata:
  creationTimestamp: 2019-01-29T07:34:37Z
  generation: 1
  name: default
  resourceVersion: "4458"
  selfLink: /apis/openebs.io/v1alpha1/storagepools/default
  uid: 54ff58b7-2398-11e9-83af-42010a8001a6
spec:
  path: /mnt/disks/ssd0/
```

Note that the `/mnt/disks/ssd0` should be formatted with `ext4` and be available
on all the nodes, where replica pods can be scheduled. 

### Configure the Replica Count

In some cases, application or the underlying storage may be doing the replication.
The number of replicas can be controlled by setting the ReplicaCount policy in 
the storage class as follows:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-jiva-r1
  annotations:
    openebs.io/cas-type: jiva
    cas.openebs.io/config: |
      - name: ReplicaCount
        value: "1"
provisioner: openebs.io/provisioner-iscsi
```


### Configure the nodes on which the target or the replica pods are scheduled.

To avoid network latencies between the application to target to replica data flow, 
it is possible to configure launching the application pods and associated target
and replica pods on the same node. 

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-jiva-r1
  annotations:
    openebs.io/cas-type: jiva
    cas.openebs.io/config: |
      - name: ReplicaCount
        value: "1"
      - name: TargetNodeSelector
        value: |-
            "kubernetes.io/hostname": "gke-kmova-perf-default-pool-8691dea0-512r"
      - name: ReplicaNodeSelector
        value: |-
            "kubernetes.io/hostname": "gke-kmova-perf-default-pool-8691dea0-512r"
provisioner: openebs.io/provisioner-iscsi
```

The application also needs to have the `nodeSelector` pointing to the same
host as the above.


## Running bench marking tests using `cStor`

cStor storage engine is the latest addition to the OpenEBS family. It differs
from the `Jiva` in the following aspects:
- Each Replica instance (called as cstor pool) can actually handle data from 
  multiple volumes. 
- The Replica instance needs to be given access to a raw block device as 
  opposed to an `ext4` mounted path.

Please refer to the https://docs.openebs.io for instantiating the cStor Pool
(aka Replica instances). 

Note: There is a cstor-sparse-pool that is launched by default as part of the
openebs installation, which will make use of sparse files from the OS disk. 
Run performance benchmark tests on cstor-sparse pool can result in OS 
disk running out of space and causing the systems to freeze. 

cStor also supports running a volume with single replica using the same configuration
like above (in `Jiva`). 

To test with cStor single replica, you can do the following:
```
#delete the defautl cstor sparse pool
kubectl delete cstor-sparse-pool
#Ensure the pool pods are cleared by running
# kubectl get pods -n openebs
# kubectl get csp
# kubectl get sp
# kubectl get spc
```

Use the following commands to setup cStor Pool with single instance. 
```
kubectl apply -f spc-cstor-sparse-1.yaml 
# A new SPC will be created - sparse1-pool
# Check the pool pods are created
# kubectl get pods -n openebs -o wide
# Note the kubernetes hostname where the pool pod is launched. 
```

Configure the StorageClass, to launch the targets on the same nodes as the pool.
```
# Edit the storageClass (sc-cstor-sparse-1.yaml) 
# Update the TargetNodeSelector with the hostname where pool is launched
kubectl apply -f sc-cstor-sparse-1.yaml
```

Update the dbench test to run on the same node as target and pool
```
# Edit the storageClass (dbench-cstor-sparse-1.yaml) 
# Update the nodeSelector with the hostname where pool is launched
kubectl apply -f dbench-cstor-sparse-1.yaml
```

Note, `TargetNodeSelector` is available from 0.8.1, if you are running 0.8.0, 
to pin the target to a node, its deployment needs to be edited
to specify the `nodeSelector`.

cStor target also allows for tuning the number of IO worker threads (called
`LU Workers`) and queue depth. For sequential workloads, having a single 
LU worker performs better. These tunables will be exposed via the storage
policies similar to the replica count in 0.9. These values are translated
into a configuration file (istgt.conf) that resides within the target pod. 
To modify them, 
 `kubectl exec -it -n openebs <cstor-target-pod> -c cstor-istgt /bin/bash`
  - cd /usr/local/istgt
  - sed -i '/Luworkers 6/c\  Luworkers 1' istgt.conf
  - check for the istgt process `ps -aux`
  - kill -9 pid <istgt-pid>
