---
title: NVIDIA GPU examples
description: >-
  Single GPU example for Hopper, Blackwell, or RTX Pro 6000 BSE 
  Multi-gpu snippets for Blackwell and Hopper; Hopper PPCIE node label
categories:
- examples
tags:
- gpu
- nvidia
---

These examples show how to request NVIDIA passthrough devices in pod specs.

They require that you have deployed CoCo following the
[NVIDIA Confidential Containers Reference Architecture](https://docs.nvidia.com/datacenter/cloud-native/confidential-containers/latest/)
which documents supported component versions and the passthrough modes used below.

In brief: NVIDIA Hopper, NVIDIA Blackwell, and NVIDIA RTX Pro 6000 all support Single-GPU passthrough (SPT).
Hopper and Blackwell additionally support Multi-GPU passthrough (MPT).
Protected PCIe (PPCIE) mode is unique to Hopper multi-gpu usages.
The following sections are example pod fragments aligned to each case.

{{% alert title="Match your hardware" color="info" %}}
Replace `runtimeClassName` with the handler for your CPU TEE (`kata-qemu-nvidia-gpu-tdx`,
`kata-qemu-nvidia-gpu-snp`, etc.).
Use `kubectl describe node <node-name>` and check Allocatable so `nvidia.com/pgpu` and
`nvidia.com/nvswitch` limits match your nodes (the values below illustrate a common eight-GPU
layout).
By default `nvidia.com/cc.mode` is `on` for confidential GPU; only the Hopper PPCIE example
below requires changing that label.
{{% /alert %}}

## 1. Hopper, Blackwell, or RTX Pro 6000 BSE: single-GPU passthrough (SPT)

No `nvidia.com/cc.mode` label change is required under default Confidential Containers / GPU
Operator settings (`on`).

Example pod requesting one GPU on Hopper, Blackwell, or RTX Pro 6000 BSE:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vectoradd-kata
  namespace: default
spec:
  runtimeClassName: kata-qemu-nvidia-gpu-tdx
  restartPolicy: Never
  containers:
  - name: cuda-vectoradd
    image: "nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda12.5.0-ubuntu22.04"
    resources:
      limits:
        nvidia.com/pgpu: "1"
```


## 2. Blackwell: multi-GPU passthrough (MPT)

Use the same pod as above, but change the resource section:

```yaml
    resources:
      limits:
        nvidia.com/pgpu: "8"
```


## 3. Hopper: multi-GPU passthrough with Protected PCIe (PPCIE) and NVSwitch

On Hopper, multi-GPU confidential passthrough uses Protected PCIe: the pod must request both GPUs
and NVSwitch devices, and the node must use `ppcie` confidential GPU mode.

```bash
kubectl label node <node-name> nvidia.com/cc.mode=ppcie --overwrite
```

Use the same pod as above, but change the resource section to include *all* node GPU resources along with their switch links to the pod:

```yaml
    resources:
      limits:
        nvidia.com/pgpu: "8"
        nvidia.com/nvswitch: "4"
```

After changing `nvidia.com/cc.mode`, wait for GPU Operator operands to settle and confirm pods are
healthy (`kubectl get pods -A`), as in the
[Kata QEMU GPU guide](https://github.com/kata-containers/kata-containers/blob/main/docs/use-cases/NVIDIA-GPU-passthrough-and-Kata-QEMU.md).

To reset for single GPU passthrough change the label back to `on`.

```bash
kubectl label node <node-name> nvidia.com/cc.mode=on --overwrite
```
