---
title: Overview
description: High-level overview of Confidential Containers
weight: 1
---

## What is the Confidential Containers project?

Confidential Containers encapsulates pods inside of confidential virtual machines,
allowing Cloud Native workloads to leverage confidential computing hardware
with minimal modification.

Confidential Containers extends the guarantees of confidential computing to complex workloads.
With Confidential Containers, sensitive workloads can be run on untrusted hosts
and be protected from compromised or malicious users, software, and administrators.

Confidential Containers provides an end-to-end framework for deploying workloads,
attesting them, and provisioning secrets.

## What hardware does Confidential Containers support?

On bare metal Confidential Containers supports the following platforms: 

| Platform | Supports Attestation |
| -------- | -------------------- |
| Intel TDX | Yes |
| AMD SEV-SNP | Yes |
| IBM Secure Execution | Yes |

### Accelerators

| Accelerator | Single Device Passthrough | Multi Device Passthrough |
| --- | --- | --- |
| Hygon DCU | Yes | No | 
| NVIDIA Hopper | Yes | Protected PCIe |
| NVIDIA RTX Pro 6000 BSE | Yes | No |
| NVIDIA Blackwell | Yes | Yes |

NVIDIA Multi-Passthrough requires assigning all GPUs on a host to one pod.
NVIDIA Protected PCIe additionally requires listing the NVIDIA NVLink switches in the pod.

### Clouds

Confidential Containers can also be deployed in a cloud environment using the
`cloud-api-adaptor`.
The following platforms are supported.

| Platform | Cloud | Notes |
| -------- | ----- | ----- |
| SNP | Azure ||
| TDX | Azure ||
| Secure Execution | IBM ||
| None | AWS | Under development |
| SNP | GCP ||
| TDX | GCP | Under development |
| None | LibVirt | For local testing |

### Attestation 

Confidential Containers provides an attestation and key-management engine, called Trustee
which is able to attest the following platforms:

| Platform |
| -------- |
| AMD SEV-SNP |
| Intel TDX |
| Intel SGX |
| AMD SEV-SNP with Azure vTPM |
| Intel TDX with Azure vTPM |
| IBM Secure Execution |
| ARM CCA | 
| Hygon CSV |
| NVIDIA GPU |

Trustee can be used with Confidential Containers or to attest standalone confidential guests.
See `Attestation` section for more information.

