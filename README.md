# AKS Managed Disk Cross-region Replication with CSI driver   




With a new feature recently added to AKS Disk CSI driver, it's possible to create a snapshot of a managed disk in one region and replicate it to a different region. Review references section links for more details.

## Pre-requisites
- VolumeSnapshotClass with remote region and resource group
- Cluster identity should be provided 'Contributor' role to the resource group 
- This resource group contains source managed disk, snapshot and remote snapshot

## References
https://github.com/kubernetes-sigs/azuredisk-csi-driver/pull/1791
 
https://learn.microsoft.com/en-us/azure/virtual-machines/disks-copy-incremental-snapshot-across-regions?tabs=azure-cli

https://azure.microsoft.com/en-us/blog/announcing-general-availability-of-incremental-snapshots-of-managed-disks/

https://azure.microsoft.com/en-us/blog/announcing-general-availability-of-incremental-snapshots-of-managed-disks/ 



### Define StorageClass

Defines resource group name, storage account type, and volume binding mode.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: scstor1
provisioner: disk.csi.azure.com
parameters:
  storageaccounttype: Standard_LRS
  kind: managed
  resourceGroup: aksrgstorage
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```


### Deploy Statefulset

Sample statefulset definition with volumeClaimTemplates. 
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  creationTimestamp: null
  labels:
    app: nginxss
  name: nginxss
spec:
  replicas: 2
  serviceName: nginxsvc
  selector:
    matchLabels:
      app: nginxss
  volumeClaimTemplates: 
  - metadata: 
      name: vol1
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 8Gi
      storageClassName: scstor1
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginxss
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
           matchLabels:
              app: nginxss
      containers:
      - image: nginx
        name: nginx
        volumeMounts: 
        - name: vol1
          mountPath: /mnt/test
```

### Define VolumeSnapshotClass

Resource group name and location are required parameters. Incremental parameter is optional. If incremental parameter is not specified, it will be set to "true" for Azure Public Cloud, and "false" for Azure Stack Cloud.  
Location should be the remote region where the snapshot will be copied to.

```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: disk-csi-snapshot-incr-remote
driver: disk.csi.azure.com
deletionPolicy: Delete
parameters:
  incremental: "true"  # available values: "true", "false" ("true" by default for Azure Public Cloud, and "false" by default for Azure Stack Cloud)
  resourceGroup: aksrgstorage
  location: westus2
```

### Create remote snapshot


This following yaml creates one volume snapshot and replicate that to remote region.   

- Create as many snapshots as needed for the volume (PVC)
- Use different names for each VolumeSnapshot  for eg. azuredisk-volume-snapshot-nginxss-1-remote-v1, azuredisk-volume-snapshot-nginxss-1-remote-v2, etc.
- Schedule VolumeSnapshot to create snapshots on a regular basis or with Velero
- Use VolumeSnapshotContent to check the status of the snapshot creation if needed

```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: azuredisk-volume-snapshot-nginxss-1-remote
spec:
  volumeSnapshotClassName: disk-csi-snapshot-incr-remote
  source: 
    persistentVolumeClaimName: vol1-nginxss-1
```


### Create PVC in remote cluster from remote snapshot

*Starting from this step, we are in a different cluster and different region same region where remote snapshot is*

#### Option 1

This requires presence of VolumeSnapshot in the remote cluster (I haven't tested this option yet)

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-azuredisk-snapshot-ss1-restored-from-snapshot
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 10Gi
  dataSource:
    name: azuredisk-volume-snapshot-nginxss-1-remote
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```


#### Option 2

CSI driver created a remote snapshot but in order to create a PVC from the remote snapshot, we need to create a disk from the remote snapshot.  This option works well and tested. 

- Create a new disk from the remote snapshot
- Create a static PV from the new disk
- Create a PVC from the static PV

Sample command to create a disk from the remote snapshot
```
az disk create --resource-group aksrgstorage --name azdisk-created-from-snapshot-v4  --sku Premium_LRS --size-gb 8 --source resourceidofsnapshot  --location westus2
```



Sample PV and PVC definition
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-recovered-from-snapshot-1
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: managed-csi
  csi:
    driver: disk.csi.azure.com
    readOnly: false
    volumeHandle: /subscriptions/102874039/resourceGroups/aksrgstorage/providers/Microsoft.Compute/disks/azdisk-created-from-snapshot
    volumeAttributes:
      fsType: ext4
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-recovered-from-snapshot-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  volumeName: pv-recovered-from-snapshot-1
  storageClassName: managed-csi
  ```

### Test restored PVC in remote cluster

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: netshoot2
  name: netshoot2
spec:
  containers:
  - args:
    - bash
    - -c
    - sleep 2000
    image: srinman/netshoot
    name: netshoot
    resources: {}
    volumeMounts: 
    - name: vol1
      mountPath: /mnt/vol1
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: vol1
    persistentVolumeClaim:
       claimName: pvc-recovered-from-snapshot-1

```


***The information provided in this blog is for general informational purposes only. The views expressed in the blog are solely those of the author and do not represent those of any company or organization. Please use this as a guidance.*** 

