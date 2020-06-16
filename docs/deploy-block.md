# Datatom infinity block CSI Plugin

The infinity block CSI plugin can enable your Container Orchestrator use datatom infinity storage.

## Overview

The infinity block CSI plugins implement an interface between CSI enabled Container Orchestrator (CO) and Infinity cluster. It allows dynamically provisioning volumes and attaching them to workloads. Current implementation of CSI plugins was tested in Kubernetes environment (requires Kubernetes 1.14+), but the code does not rely on any Kubernetes specific calls (WIP to make it k8s agnostic) and should be able to run with any CSI enabled CO.

Before to go,you should have installed [Infinity3.5](http://www.datatom.com/cn/productions/frame/infinity/).

You can get latest version of DATATOM infinity block CSI driver at [docker hub](https://hub.docker.com/r/datatom/infinitycsi/tags).

# Deployment with Kubernetes

In this section,you will learn how to deploy the CSI driver and some necessary sidecar containers.

## Prepare cluster ##

|   Cluster   |   version   |
| ----------- | ----------- |
|  Kubernetes |    1.14+    |
|   Infinity  |    3.5+     |


Your Kubernetes cluster must allow privileged pods (i.e. `--allow-privileged`
flag must be set to true for both the API server and the kubelet). Moreover, as
stated in the [mount propagation
docs](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation),
the Docker daemon of the cluster nodes must allow shared mounts.

The deploy YAML manifests are located in [deploy/block/kubernetes](https://github.com/datatom-infinity/infinity-csi/blob/master/deploy/block/kubernetes).

**Deploy RBACs for sidecar containers and node plugins:**

```bash
kubectl create -f csi-infinity-provisioner-rbac.yaml
kubectl create -f csi-infinity-nodeplugin-rbac.yaml
```

**Deploy PodSecurityPolicy resources for sidecar containers and node plugins:**

**NOTE:** These manifests are required only if [PodSecurityPolicy](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy)
admission controller is active on your cluster.

```bash
kubectl create -f csi-infinity-provisioner-psp.yaml
kubectl create -f csi-infinity-nodeplugin-psp.yaml
```

**Deploy ConfigMap for CSI plugins:**

```bash
kubectl create -f csi-infinity-config-map.yaml
```

Following are examples for csi-infinity-config-map.yaml:

```yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "2176d19b-dbf7-45e0-9eff-8b4b04946f7f",
        "monitors": [
          "10.0.7.35",
          "10.0.7.52",
          "10.0.7.73"
        ]
      }
    ]
metadata:
  name: infinity-csi-config
```

The config map can be updated in the cluster using the following command:

```bash
kubectl replace -f csi-infinity-config-map.yaml
```

**Deploy CSI sidecar containers:**

```bash
kubectl create -f csi-infinity-blockplugin-provisioner.yaml
```

Deploys stateful set of provision which includes external-provisioner
,external-attacher,csi-snapshotter sidecar containers and CSI plugin.

**Deploy Block CSI driver:**

```bash
kubectl create -f csi-infinity-blockplugin.yaml
```

Deploys a daemon set with two containers: CSI node-driver-registrar and the CSI
block driver.

## Verifying the deployment in Kubernetes

After successfully completing the steps above, you should see output similar to this:

```bash
$ kubectl get all
NAME                                               READY   STATUS    RESTARTS   AGE
pod/csi-blockplugin-mdhnf                          3/3     Running   0          10s
pod/csi-blockplugin-provisioner-7d59684b88-2nh67   6/6     Running   0          10s
pod/csi-blockplugin-provisioner-7d59684b88-f75g9   6/6     Running   0          10s
pod/csi-blockplugin-provisioner-7d59684b88-pbsbl   6/6     Running   0          10s

NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/csi-blockplugin-provisioner   ClusterIP   10.100.49.156   <none>        8080/TCP,8090/TCP   13s
...
```

# Test block plugin
The test YAML manifests are located in [test](https://github.com/datatom-infinity/infinity-csi/blob/master/test/infiblock/kubernetes/).

## Deploying the storage class

Once the plugin is successfully deployed, you'll need to customize
`storageclass.yaml` and `secret.yaml` manifests to reflect your infinity cluster
setup.
Please consult the documentation for info about available parameters.

After configuring the secrets, monitors, etc. you can deploy a
testing Pod mounting a volume:

```bash
kubectl create -f secret.yaml
kubectl create -f storageclass.yaml
kubectl create -f pvc.yaml
kubectl create -f pod.yaml
```

## How to test Snapshot feature

Before continuing, make sure you enabled the required
feature gate `VolumeSnapshotDataSource=true` in your Kubernetes cluster.

In the `test` directory you will find two files related to snapshots:
[snapshotclass.yaml](https://github.com/datatom-infinity/infinity-csi/blob/master/test/infiblock/kubernetes/snapshotclass.yaml) and
[snapshot.yaml](https://github.com/datatom-infinity/infinity-csi/blob/master/test/infiblock/kubernetes/snapshot.yaml).

Once you created your volume, you'll need to customize at least
`snapshotclass.yaml` and make sure the `clusterid` parameter matches
your infinity cluster setup.
If you followed the documentation to create the blockplugin, you shouldn't
have to edit any other file.

After configuring everything you needed, deploy the snapshot class:

```bash
kubectl create -f snapshotclass.yaml
```

Verify that the snapshot class was created:

```console
$ kubectl get volumesnapshotclass
NAME                        AGE
csi-blockplugin-snapclass   4s
```

Create a snapshot from the existing PVC:

```bash
kubectl create -f snapshot.yaml
```

To verify if your volume snapshot has successfully been created, run the following:

```console
$ kubectl get volumesnapshot
NAME                 AGE
block-pvc-snapshot   6s
```

To check the status of the snapshot, run the following:

```bash
$ kubectl describe volumesnapshot block-pvc-snapshot
```

To be sure everything is OK you can check it in the infinity admin interface.
![snapshot_info](https://github.com/datatom-infinity/infinity-csi/blob/master/images/block/snapshot.png)

To restore the snapshot to a new PVC, deploy
[pvc-restore.yaml](https://github.com/datatom-infinity/infinity-csi/blob/master/test/infiblock/kubernetes/pvc-restore.yaml) and a testing pod
[pod-restore.yaml](https://github.com/datatom-infinity/infinity-csi/blob/master/test/infiblock/kubernetes/pod-restore.yaml):

```bash
kubectl create -f pvc-restore.yaml
kubectl create -f pod-restore.yaml
```

