---
title: SEV-SNP Host Setup 
description: Host configurations for AMD SEV-SNP machines 
weight: 10
categories:
- prerequisites
tags:
- SEV
---

## Platform Setup

The host BIOS and kernel must be capable of supporting AMD SEV-SNP and the host must be configured accordingly.

The SEV Firmware version must be at least version 1.55 in order to have at least version 3 of the Attestation Report. The latest SEV Firmware version is available on AMD's [SEV Developer Webpage](https://www.amd.com/en/developer/sev.html). It can also be updated via a platform OEM BIOS update.

The host kernel must be equal to or later than upstream version [6.16.1](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git).

To build just the upstream compatible host kernel, use the Confidential Containers fork of [AMDESE AMDSEV](https://github.com/confidential-containers/amdese-amdsev/tree/amd-snp-202501150000). Individual components can be built by running the following command:

```
./build.sh kernel host --install
```