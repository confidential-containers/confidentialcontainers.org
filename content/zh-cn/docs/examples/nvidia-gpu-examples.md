---
title: NVIDIA GPU 示例
description: >-
  Hopper、Blackwell 与 RTX Pro 6000 BSE 的单 GPU 示例；
  Blackwell 与 Hopper 的多 GPU 示例；Hopper PPCIE 节点标签说明
categories:
- examples
tags:
- gpu
- nvidia
---

> **说明：** 本文为英文文档的中文译版，英文原版请参见 [NVIDIA GPU examples](https://confidentialcontainers.org/docs/examples/nvidia-gpu-examples/)。

本文展示如何在 Pod 中配置 NVIDIA 设备直通。

配置前提是你已经按照 [NVIDIA Confidential Containers Reference Architecture](https://docs.nvidia.com/datacenter/cloud-native/confidential-containers/latest/) 部署了 CoCo，并且所用组件版本与下文的直通模式相匹配。若你想看一个把 GPU runtime class、Trustee/KBS、sealed secret、guest pull 和生成的 agent policy 串起来的完整 NIM 场景，可参考 [带 GPU 远程证明的 NVIDIA NIM 部署场景](https://confidentialcontainers.org/docs/examples/nvidia-nim-confidential-gpu-attestation/)。

简要说明如下：NVIDIA Hopper、NVIDIA Blackwell 和 NVIDIA RTX Pro 6000 都支持单 GPU 直通（SPT）。Hopper 和 Blackwell 另外支持多 GPU 场景（ PPCIE 和 MPT ）。Protected PCIe（PPCIE）只用于 Hopper 多 GPU，要求 NVLink switch 进入 PPCIE 模式，CoCo 会对齐进行配置。Multi-GPU Passthrough（MPT）用于 Blackwell GPU 之间的端到端加密。NVIDIA RTX Pro 不支持多 GPU 配置。下面分别给出各场景下 Pod 的配置。

{{% alert title="按硬件调整" color="info" %}}
请根据你的 CPU TEE 类型选择相应的 `runtimeClassName`，例如 `kata-qemu-nvidia-gpu-tdx`、`kata-qemu-nvidia-gpu-snp`。
用 `kubectl describe node <node-name>` 查看 Allocatable，确认 `nvidia.com/pgpu` 和 `nvidia.com/nvswitch` 的资源上限与节点实际能力一致。下面的数值以常见的 8 GPU 节点为例。
默认情况下，机密 GPU 使用 `nvidia.com/cc.mode=on`；只有下面的 Hopper PPCIE 示例需要改这个标签。
{{% /alert %}}

## 1. Hopper、Blackwell 或 RTX Pro 6000 BSE：单 GPU 直通（SPT）

在默认的 Confidential Containers / GPU Operator 配置下（`on`），这里不需要修改 `nvidia.com/cc.mode`。

下面的 Pod 示例会在 Hopper、Blackwell 或 RTX Pro 6000 BSE 上申请 1 张 GPU：

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

## 2. Blackwell（B200、B300）：多 GPU 直通（MPT）

Pod 其他部分不变，只需要把资源配置改成：

```yaml
    resources:
      limits:
        nvidia.com/pgpu: "8"
```

## 3. Hopper：带 Protected PCIe（PPCIE）和 NVSwitch 的多 GPU 直通

在 Hopper 上，多 GPU 机密直通使用 Protected PCIe。也就是说，Pod 必须同时申请 GPU 和 NVSwitch 设备，节点也必须切到 `ppcie` 机密 GPU 模式。

```bash
kubectl label node <node-name> nvidia.com/cc.mode=ppcie --overwrite
```

Pod 其他部分仍然保持不变，只需要把资源段改成如下配置，把节点上的 GPU 与 NVSwitch 分配给 Pod：

```yaml
    resources:
      limits:
        nvidia.com/pgpu: "8"
        nvidia.com/nvswitch: "4"
```

修改 `nvidia.com/cc.mode` 之后，等待 GPU Operator 相关组件重新稳定，并确认 Pod 状态正常（`kubectl get pods -A`）。可参考 [Kata QEMU GPU guide](https://github.com/kata-containers/kata-containers/blob/main/docs/use-cases/NVIDIA-GPU-passthrough-and-Kata-QEMU.md) 的做法。

如果要恢复为单 GPU 直通，把标签改回 `on`：

```bash
kubectl label node <node-name> nvidia.com/cc.mode=on --overwrite
```
