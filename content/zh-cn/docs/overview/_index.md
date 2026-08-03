---
title: 概览
description: Confidential Containers 概览
weight: 1
---

## 什么是 Confidential Containers（CoCo）项目？

Confidential Containers（CoCo）通过将 Pod 运行在机密虚拟机（Confidential Virtual Machine）中，
让云原生工作负载在几乎无需修改的情况下利用机密计算硬件提供的安全能力。

Confidential Containers 将机密计算保护能力扩展到复杂工作负载场景。
借助 Confidential Containers，敏感工作负载即便运行在不受信任的主机环境（untrusted hosts）上，
也能够抵御来自已遭入侵或恶意的用户、软件和管理员的攻击。

Confidential Containers 提供一个 **工作负载部署**、**远程证明（Remote Attestation）** 以及 **密钥/机密信息（Keys/Secrets）分发** 的端到端框架。

## Confidential Containers 支持哪些硬件？

在裸金属环境下，Confidential Containers 支持以下平台：

| 平台 | 支持远程证明（Remote Attestation） |
| -------- | -------------------- |
| Intel TDX | 是 |
| AMD SEV-SNP | 是 |
| IBM Secure Execution | 是 |

### 硬件加速器（Accelerators）

| 硬件加速器 | 单设备透传 | 多设备透传 |
| --- | --- | --- |
| Hygon DCU | 是 | 否 |
| NVIDIA Hopper | 是 | Protected PCIe |
| NVIDIA RTX Pro 6000 BSE | 是 | 否 |
| NVIDIA Blackwell | 是 | 是 |

NVIDIA 多设备透传要求将主机上的全部 GPU 分配给同一个 Pod。
NVIDIA Protected PCIe 还要求在 Pod 中列出 NVIDIA NVLink 交换机。

### 云平台

还可以借助`cloud-api-adaptor`在**云环境**中部署Confidential Containers。

目前支持以下平台。

| 平台 | 云 | 备注 |
| -------- | ----- | ----- |
| SNP | Azure ||
| TDX | Azure ||
| TDX | 阿里云 ||
| Secure Execution | IBM ||
| None | AWS | 开发中 |
| SNP | GCP ||
| TDX | GCP | 开发中 |
| None | LibVirt | 用于本地测试 |

### 远程证明（Attestation）

Confidential Containers 提供了一个名为 Trustee 的远程证明与密钥管理引擎，可为以下平台提供远程证明：

| 平台 |
| -------- |
| AMD SEV-SNP |
| Intel TDX |
| Intel SGX |
| AMD SEV-SNP with Azure vTPM |
| Intel TDX with Azure vTPM |
| IBM Secure Execution |
| ARM CCA |
| Hygon CSV |
| Hygon DCU |
| NVIDIA GPU |

Trustee 既可以与 Confidential Containers 配合使用，也可以用于对独立运行的机密虚拟机（即非 CoCo 场景）进行远程证明。
更多信息请参见“远程证明（Attestation）”章节。
