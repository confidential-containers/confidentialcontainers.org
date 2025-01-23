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

| Platform | Supports Attestation | Uses Kata |
| -------- | -------------------- | --------- |
| Intel TDX | Yes | Yes |
| Intel SGX | Yes | No |
| AMD SEV-SNP | Yes | Yes |
| AMD SEV(-ES) | No | Yes |
| IBM Secure Execution | Yes | Yes |

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

Trustee can be used with Confidential Containers or to attest standalone confidential guests.
See `Attestation` section for more information.

