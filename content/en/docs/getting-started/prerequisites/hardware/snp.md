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

In order to launch SNP memory encrypted guests, the host must be prepared with a compatible kernel, `6.8.0-rc5-next-20240221-snp-host-cc2568386`. AMD custom changes and required components and repositories will eventually be taken upstream. 

[Sev-utils](https://github.com/amd/sev-utils/blob/coco-202402240000/docs/snp.md) is an easy way to install the required host kernel, but it will unnecessarily build AMD compatible guest kernel, OVMF, and QEMU components. The additional components can be used with the script utility to test launch and attest a base QEMU SNP guest. However, for the CoCo use case, make sure to use the coco tagged version because they are already packaged and delivered with Kata.

Alternatively, refer to the [AMDESE guide](https://github.com/confidential-containers/amdese-amdsev/tree/amd-snp-202402240000?tab=readme-ov-file#prepare-host) to manually build the host kernel and other components.

### Create the Certificate-Chain

Certificate chains (cert-chain) are an essential piece for creating a secure and trusted environment, ensuring that data is processed securely and that all components involved are verified and trustworthy. Use the following commands to create a cert-chain for SNP using snphost.

```
git clone https://github.com/virtee/snphost.git && cd snphost/
cargo build
mkdir /tmp/certs
./target/debug/snphost fetch vcek der /tmp/certs
./target/debug/snphost import /tmp/certs /opt/snp/cert_chain.cert
```

### Edit the SNP Config File

In order to use cert chain that was made for SNP, the Kata SNP configuration file needs to be edited first. The config file can be found under ``` /opt/kata/share/defaults/kata-containers/configuration-qemu-snp.toml ```.

The location of the certificate chain needs to be specified under ```snp_certs_path```.

```
snp_certs_path = "{PATH TO cert_chain.cert}"
```