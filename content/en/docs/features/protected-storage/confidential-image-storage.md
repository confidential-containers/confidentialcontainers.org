---
title: Confidential Container Image Storage
description: "Experimental: Use protected ephemeral storage instead of confidential guest memory for guest-pulled image layers"
weight: 40
categories:
- feature
tags:
- storage
---

With [guest image management](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-pull-images-in-guest-with-kata.md),
container images are pulled and unpacked inside the confidential guest.
Like other files written to the workload filesystem, their layers are stored
in confidential guest memory by default. Specifically, the layers are written
to `/run/kata-containers/image`, which is backed by guest memory. Large images
can therefore significantly increase a pod's memory requirements.

This experimental storage feature moves these layers to a host-provided block
device.
The Kata agent asks the Confidential Data Hub (CDH) to encrypt and
integrity-protect the device inside the guest using dm-crypt and dm-integrity,
then mount it at `/run/kata-containers/image`.
The host provides the storage but cannot read or modify individual data blocks
without detection.

{{% alert title="Warning" color="warning" %}}
Confidential image storage is ephemeral and does not provide replay protection.
Both downloaded image layers and the OverlayFS upper and work directories for
the container writable layer are stored on the protected block device.
{{% /alert %}}

Provide a raw block PersistentVolumeClaim using a suitable storage driver and
expose it to a container at the reserved device path `/dev/trusted_store`. The
PersistentVolumeClaim must use `volumeMode: Block`. The Kata agent detects the
reserved device path and prepares the device for image-layer storage.

```yaml
spec:
  volumes:
    - name: image-storage
      persistentVolumeClaim:
        claimName: image-storage-pvc
  containers:
    - name: app
      volumeDevices:
        - name: image-storage
          devicePath: /dev/trusted_store
```

From the workload's perspective, this storage is ephemeral: it is not an image
cache that can be reused by later pods. The lifecycle of the underlying
PersistentVolume still follows its Kubernetes reclaim policy. The feature
reduces confidential guest memory usage for container image layers.

See the Kata Containers guide for the complete
[guest image management and block-volume setup](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-pull-images-in-guest-with-kata.md#create-block-volume-with-k8s).

## Try it with local storage

For prototyping on a single node, a file on a disk-backed host filesystem can
be attached as a loop device and exposed through a local PersistentVolume.
This setup requires root access to the node and is not suitable for production.

Create a sparse 64 GiB backing file and attach it to an available loop device.
The host filesystem must have enough free space as the file grows:

```bash
NODE_NAME="$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"
LOOP_FILE="/var/lib/coco/image-storage.img"

sudo mkdir -p "$(dirname "${LOOP_FILE}")"
sudo truncate --size 64G "${LOOP_FILE}"
LOOP_DEVICE="$(sudo losetup --find --show "${LOOP_FILE}")"
```

Create a StorageClass, local PersistentVolume, and raw block
PersistentVolumeClaim for that device:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-image-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: image-storage-pv
spec:
  capacity:
    storage: 64Gi
  volumeMode: Block
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-image-storage
  local:
    path: ${LOOP_DEVICE}
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - ${NODE_NAME}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-storage-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 64Gi
  volumeMode: Block
  storageClassName: local-image-storage
EOF
```

The PVC can remain `Pending` until the pod is created because the StorageClass
uses `WaitForFirstConsumer`. Attach the claim to an existing guest-pull workload
at `/dev/trusted_store`.

After deleting the workload, remove the storage objects, detach the loop
device, and remove the backing file:

```bash
kubectl delete pvc image-storage-pvc
kubectl delete pv image-storage-pv
kubectl delete storageclass local-image-storage
sudo losetup --detach "${LOOP_DEVICE}"
sudo rm -f "${LOOP_FILE}"
```
