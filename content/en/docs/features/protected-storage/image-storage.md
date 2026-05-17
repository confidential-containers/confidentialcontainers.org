---
title: Image Storage 
description: Protected storage for container images 
weight: 30
categories:
- feature 
tags:
- storage 
---

Container images can be pulled into an encrypted block device.
This is useful for pulling large images without exhausting guest memory.
Today, only ephemeral block devices are supported, so these images cannot
be shared between pods.

{{% alert title="Warning" color="warning" %}}
While confidential image storage does have integrity protection via LUKS2,
it does not have replay protection.
While the host cannot modify individual blocks, it can rollback the entire
volume to some earlier state.
{{% /alert %}}

Protected image storage can be built on any block volume,
but this example will show Local Persistent Volumes.
This example is not applicable to peer pods.

## Setup

Create a disk image attached to a loop device.

```console
$ loop_file="/tmp/trusted-image-storage.img"
$ sudo dd if=/dev/zero of=$loop_file bs=1M count=2500
$ sudo losetup /dev/loop0 $loop_file
```

Create a storage class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Create a persistent volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: trusted-block-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Block
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /dev/loop0
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - NODE_NAME
```

Create a persistent volume claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: trusted-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeMode: Block
  storageClassName: local-storage
```

Run a pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: large-image-pod
spec:
  runtimeClassName: kata-qemu-coco-dev
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - NODE_NAME
  volumes:
    - name: trusted-image-storage
      persistentVolumeClaim:
        claimName: trusted-pvc
  containers:
    - name: app-container
      image: quay.io/confidential-containers/test-images:largeimage
      command: ["/bin/sh", "-c"]
      args:
        - sleep 6000
      volumeDevices:
        - devicePath: /dev/trusted_store
          name: trusted-image-storage
```
