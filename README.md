# Installing

This tutorial can be used to deploy Rook+Ceph on OpenShift 4.1. Assumes installation on AWS but these instructions have also been tested with RHV and VMware based OCP deployments.

```console
$ git clone https://github.com/rook/rook.git
```

We will use the Rook operator to deploy Ceph. The following yaml files are used to create CRDs, RoleBindings and other objects needed to support Rook and the Ceph cluster.

```console
$ cd cluster/examples/kubernetes/ceph
$ oc create -f common.yaml
$ oc create -f crds.yaml
```

Before proceeding we need to create EBS volumes for Ceph to use. These volumes will be presented as raw block devices to the nodes. My reference cluster is deployed on AWS and uses 3x m5.2xlarge worker nodes. I added 1x400 EBS volume to each node. For other platforms, just ensure raw (unformatted/unpartitioned) devices are presented to the nodes you want to deploy Ceph on. If you opt for the default installation, discovery pods will be deployed to automatically enumerate available storage.

Next, create the operator (verify it is running by executing `oc get pods -n rook-ceph | grep operator`). Then create the cluster by sourcing `cluster.yaml`.

```console
$ oc create -f operator-openshift.yaml
$ oc create -f cluster.yaml
```

Verify the deployment of the cluster by running `oc get pods -n rook-ceph` as shown below.

```console

$ oc get pods -n rook-ceph
NAME                                          READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-24ghk                        2/2     Running     0          17m
csi-cephfsplugin-gs42d                        2/2     Running     0          17m
csi-cephfsplugin-lmjss                        2/2     Running     0          17m
csi-cephfsplugin-provisioner-0                2/2     Running     0          17m
csi-rbdplugin-fsf6l                           2/2     Running     0          17m
csi-rbdplugin-jl84w                           2/2     Running     0          17m
csi-rbdplugin-mt9xc                           2/2     Running     0          17m
csi-rbdplugin-provisioner-0                   4/4     Running     0          17m
rook-ceph-agent-84g4n                         1/1     Running     0          17m
rook-ceph-agent-gmjq6                         1/1     Running     0          17m
rook-ceph-agent-hljgk                         1/1     Running     0          17m
rook-ceph-mgr-a-649ccccddb-494r5              1/1     Running     0          3m21s
rook-ceph-mon-a-d746fd87b-phr7w               1/1     Running     0          4m20s
rook-ceph-mon-b-7f959f6fc7-445g9              1/1     Running     0          4m7s
rook-ceph-mon-c-57f698c8b4-4ntp8              1/1     Running     0          3m46s
rook-ceph-operator-fbdb6784c-zcns5            1/1     Running     0          17m
rook-ceph-osd-0-f8c5bb5d9-xsjkq               1/1     Running     0          2m19s
rook-ceph-osd-1-78d4875965-76khn              1/1     Running     0          2m19s
rook-ceph-osd-2-55cd9ffc5c-d9d74              1/1     Running     0          2m19s
rook-ceph-osd-prepare-ip-10-0-204-150-6fjfj   0/2     Completed   0          2m54s
rook-ceph-osd-prepare-ip-10-0-206-173-w8qzk   0/2     Completed   0          2m54s
rook-ceph-osd-prepare-ip-10-0-215-44-sf8t6    0/2     Completed   0          2m54s
rook-discover-fp68m                           1/1     Running     0          17m
rook-discover-h49qs                           1/1     Running     0          17m
rook-discover-wr69x                           1/1     Running     0          17m
```

Next we create a CephFilesystem object using the Rook operator. We also create a debug pod to interface with the deployed Ceph cluster.

```console
$ oc create -f filesystem.yaml
$ oc create -f toolbox.yaml
```

Execute a shell on the debug pod to get the status of the Ceph cluster.

```console
$ oc -n rook-ceph exec -it $(oc -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
```

Then run `ceph status` and observe the output.

```console

# ceph status
  cluster:
    id:     d1a84232-f042-4e50-8541-051456744d7a
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 9m)
    mgr: a(active, since 9m)
    mds: myfs:1 {0=myfs-b=up:active} 1 up:standby-replay
    osd: 3 osds: 3 up (since 8m), 3 in (since 8m)
 
  data:
    pools:   2 pools, 200 pgs
    objects: 22 objects, 2.2 KiB
    usage:   3.0 GiB used, 1.2 TiB / 1.2 TiB avail
    pgs:     200 active+clean
 
  io:
    client:   853 B/s rd, 1 op/s rd, 0 op/s wr
```

Finally we need to create the StorageClass objects for CephFS and RBD.

## Storage Class for csi-cephfs

```console
$ oc create -f csi/cephfs/storageclass.yaml
```

## Storage Class for csi-rbd

```console
$ oc create -f csi/rbd/storageclass.yaml
```

When you are finished verify the StorageClass objects were created as shown:

```console
$ oc get sc
NAME              PROVISIONER           AGE
csi-cephfs        cephfs.csi.ceph.com   8m36s
rook-ceph-block   rbd.csi.ceph.com      9m29s
...

```

# Verifying

To verify basic functionality, create a project to test the create of some PVCs.

```console
$ oc new-project csi-test
```

## Testing CephFS

```console
$ cd rook/cluster/examples/kubernetes/ceph
$ oc create -f csi/cephfs/pvc.yaml -n csi-test
$ oc create -f csi/cephfs/pod.yaml -n csi-test
```

Verify the CephFS PVC and test pod as follows:

```console
$ oc get pvc cephfs-pvc -n csi-test
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cephfs-pvc   Bound    pvc-0ce77415-b3aa-11e9-9478-123b081d1798   1Gi        RWO            csi-cephfs     12s
```

```console
$ oc get pod csicephfs-demo-pod -n csi-test
NAME                 READY   STATUS    RESTARTS   AGE
csicephfs-demo-pod   1/1     Running   0          26s
```

## Testing RBD

First, modify the contents of csi/cephfs/pvc.yaml and change the storageClassName from rook-csi to rook-ceph-block. Then run the following commands:

```console
$ cd rook/cluster/examples/kubernetes/ceph
$ oc create -f csi/rbd/pvc.yaml -n csi-test
$ oc create -f csi/rbd/pod.yaml
```

Verify the RBD PVC and test pod as follows:

```console
$ oc get pvc rbd-pvc -n csi-test
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rbd-pvc   Bound    pvc-a53e879c-b3aa-11e9-9478-123b081d1798   1Gi        RWO            csi-rbd        19s
```

```console
$ oc get pods csirbd-demo-pod -n csi-test
NAME              READY   STATUS    RESTARTS   AGE
csirbd-demo-pod   1/1     Running   0          66s
```

# Configuring OpenShift Logging to Use Ceph RBD

As an extra level of validation we can deploy the OpenShift EFK stack and configure Elasticsearch PVCs to use the csi-rbd storage class.

To do this, follow the instructions referenced in the beginning of the document. When you get to step five (5. Create a cluster logging instance:) use the following yaml:

```yaml
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance"
  namespace: "openshift-logging"
spec:
  managementState: "Managed"
  logStore:
    type: "elasticsearch"
    elasticsearch:
      nodeCount: 3
      storage:
      	# storage class is modified here
        storageClassName: rook-ceph-block
        size: 100G
      redundancyPolicy: "SingleRedundancy"
  visualization:
    type: "kibana"
    kibana:
      replicas: 1
  curation:
    type: "curator"
    curator:
      schedule: "30 3 * * *"
  collection:
    logs:
      type: "fluentd"
      fluentd: {}
```

Continue with the official documentation to complete the install. After install, verify the Elasticsearch pods are using storage provided by the csi-rbd storage class:

```console
$ oc get pvc -n openshift-logging
NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-elasticsearch-cdm-b68hqpif-1   Bound    pvc-3829d7b7-b3a1-11e9-84c0-0206e4668600   94Gi       RWO            csi-rbd        3h9m
elasticsearch-elasticsearch-cdm-b68hqpif-2   Bound    pvc-3847ea96-b3a1-11e9-84c0-0206e4668600   94Gi       RWO            csi-rbd        3h9m
elasticsearch-elasticsearch-cdm-b68hqpif-3   Bound    pvc-386668ad-b3a1-11e9-84c0-0206e4668600   94Gi       RWO            csi-rbd        3h9m
```

# Appendix
## install-config.yaml

```yaml
apiVersion: v1
baseDomain: redhat-demo.com
controlPlane:
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      zones:
      - us-east-1a
      - us-east-1b
      - us-east-1c
      type: m5.xlarge
  replicas: 3
compute:
- hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: m5.2xlarge
      zones:
      - us-east-1d
      # WTF us-east-1e doesn't support m5.2xlarge?
      # - us-east-1e
      - us-east-1f
  replicas: 3
metadata:
  name: ckeller-aws
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineCIDR: 10.0.0.0/16
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: us-east-1
    userTags:
      adminContact: ckeller@redhat.com
pullSecret: ‘’
sshKey: ‘’
```
