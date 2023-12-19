---
title: About Confidential Containers
linkTitle: About
menu:
  main:
    weight: 10
---

{{% blocks/cover image_anchor="bottom" height="auto" %}}
<p class="fw-bold fa-3x">
About Confidential Containers
</p>
{{% /blocks/cover %}}

{{% blocks/section %}}

## Confidential Containers - WHAT

<br>

- [Confidential Containers](https://github.com/confidential-containers) (CoCo) is a sandbox project in [Cloud Native Computing Foundation](https://www.cncf.io/) (CNCF)

- It enables cloud-native [confidential computing](https://confidentialcomputing.io/faq/) by taking advantage of a variety of hardware platforms and technologies

- The CoCo project builds on existing and emerging hardware security technologies such as Intel SGX, Intel TDX, AMD SEV and IBM Z Secure Execution, in combination with new software frameworks **to protect data in use**

- The project brings together software and hardware companies including Alibaba-cloud, AMD, ARM, IBM, Intel, Microsoft, Red Hat, Rivos, Edgeless Systems and others

{{% /blocks/section %}}

{{% blocks/section %}}

## Confidential Containers - WHY

<br>

- A [Trusted Execution Environments](https://en.wikipedia.org/wiki/Trusted_execution_environment) (TEE) is at the heart of a confidential computing solution
  - _TEEs are isolated environments with enhanced security, provided by confidential computing (CC) capable hardware that prevents unauthorized access or modification of applications and data while in use_

- The CoCo project integrates TEE infrastructure with the cloud-native world

- The **goal** of CoCo is to **standardize** confidential computing at the container level and **simplify** its consumption in Kubernetes

- This is in order to enable Kubernetes users to deploy confidential container workloads using familiar workflows and tools without extensive knowledge of underlying confidential computing technologies

{{% /blocks/section %}}

{{% blocks/section %}}

## Confidential Containers - HOW

<br>

- CoCo enables you to deploy your workload on infrastructure owned by someone else

- The infrastructure can be managed by a cloud provider, a different division in your organization such as the IT department or even an untrusted third party

- This is achieved by encrypting your workload memory and protecting other low level resources the workload requires at the hardware level

- Cryptography-based proofs is used to confirm that your software runs without being tampered with or fails your workload from running if that isn’t the case

{{% /blocks/section %}}

{{% blocks/section %}}

## A project which aims to be usable

<br>

- The project puts a strong emphasis on delivering practical cloud native solution:

  - **Simplicity** - Using a dedicated Kubernetes operator for deployment and configuration. Making this technology as accessible as possible hiding away most of the hardware-dependent parts

  - **Stability** - Supporting continuous integration (CI) for the key workflows of the release

  - **Use case driven development** - focusing the community around a few key use cases including supporting CI/CD instead of feature based development

{{% /blocks/section %}}
