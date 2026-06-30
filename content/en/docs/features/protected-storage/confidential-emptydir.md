---
title: Confidential EmptyDir
description: Protected ephemeral storage for workloads 
weight: 30
categories:
- feature 
tags:
- storage 
---

When using a confidential runtime class, all emptyDir volumes 
will automatically be created on top of secure block devices.
These confidential emptyDir volumes use LUKS2 on top of a block device provided by the host.

Confidential emptyDir can be a good fit for a workload that needs to write
a lot of secret data to a scratch directory.
If stored inside the guest, this data could deplete guest memory.
Instead, the confidential emptyDir is backed by a block device provided by the host.
The block device is encrypted inside the guest such that the host cannot access the data.

{{% alert title="Warning" color="warning" %}}
While confidential emptyDir volumes have integrity protection via LUKS2,
they do not have replay protection.
While the host cannot modify individual blocks, they can rollback the entire
volume to some earlier state.
{{% /alert %}}

A confidential emptyDir can be added to a workload the same way a traditional
emptyDir would be used.

```yaml
volumeMounts:
      - name: scratch-volume 
        mountPath: /scratch-directory
  volumes:
  - name: scratch-volume
    emptyDir:
      sizeLimit: 64Gi
```

On the host, this volume will be backed by a sparse file. Data blocks are allocated as the volume
is used, but creating the filesystem, encryption, and integrity metadata consumes some physical
host storage immediately.

{{% alert title="Sizing block-backed emptyDir volumes" color="warning" %}}
Kata Containers does not currently use `emptyDir.sizeLimit` to size the block device presented to
the guest. Instead, it sets the maximum size of the sparse backing image, and therefore the logical
capacity of the guest block device, to the capacity of the host filesystem containing the volume.
Kubelet still counts the physically allocated blocks against `sizeLimit`.

With the current ext4 formatting options, initial filesystem metadata uses approximately 0.08% of
the logical capacity (about 2.5 GB for 2.9 TB). Encryption and integrity added about 50 MB in the
same test.

On a large host filesystem, the initial metadata allocation can consume a significant part of a
small `sizeLimit`, reducing the space available to the workload or causing the pod to be evicted
before it writes data. Allow enough headroom in `sizeLimit` for this metadata, or place kubelet's
volume data on a smaller dedicated filesystem. See the Kata Containers
[storage limitations](https://github.com/kata-containers/kata-containers/blob/main/docs/Limitations.md#block-backed-kubernetes-emptydir-volumes)
for current overhead estimates, implementation details, and tracked improvements.
{{% /alert %}}

{{% alert title="Note" color="note" %}}
LUKS2 does not support discard/trim when integrity is enabled.
Thus, when a file is removed from the emptyDir by the guest,
the size of the sparse file will not decrease.
Over the lifespan of the workload, the size of the sparse file
can only increase.
{{% /alert %}}

Confidential emptyDir volumes are ephemeral. They are removed when the pod is torn down.

The LUKS2 header for the volume is stored in guest memory and is not accessible to the host.

If you want to use an emptyDir that isn't backed by a LUKS volume,
set the emptyDir `medium` to `Memory`. This will create an emptyDir
that is stored in guest memory.

```yaml
volumeMounts:
  - name: memory-empty-vol
    mountPath: "/tmp/cache"
volumes:
  - name: memory-empty-vol
  emptyDir:
    medium: Memory
    sizeLimit: "50M"
```
