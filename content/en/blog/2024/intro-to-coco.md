---
date: 2024-02-16
title: Introduction to Confidential Containers (CoCo)
linkTitle: Introduction to Confidential Containers (CoCo)
description: >
  Introduction to the project: Motivation, Mechanics, Foundational Principles and Community.
author: Suraj Deshmukh ([@surajd_](https://twitter.com/surajd_)) & [Ariel Adam](https://www.linkedin.com/in/ariel-adam-26398b55/)
---

{{% alert color="info" %}}
This blog is adopted from the [overview slides](https://docs.google.com/presentation/d/1cMiehbYq5vRcSwZ0kp_VMCr687ochBtblFFvtx1pl7E/edit?usp=sharing) on Confidential Containers.
{{% /alert %}}

![](/img/confidential-containers-color.svg)

**[Confidential Containers](https://github.com/confidential-containers) (CoCo)** is an innovative sandbox project under the [Cloud Native Computing Foundation](https://www.cncf.io/) (CNCF), revolutionizing cloud-native [confidential computing](https://confidentialcomputing.io/faq/) by leveraging diverse hardware platforms and cutting-edge technologies.

The CoCo project builds on existing and emerging hardware security technologies such as [Intel SGX](https://en.wikipedia.org/wiki/Software_Guard_Extensions), [Intel TDX](https://en.wikipedia.org/wiki/Trust_Domain_Extensions), [AMD SEV-SNP](https://www.amd.com/en/processors/amd-secure-encrypted-virtualization) and [IBM Z Secure Execution](https://en.wikipedia.org/wiki/IBM_Secure_Service_Container), in combination with new software frameworks to protect **data in use**. The project brings together software and hardware companies including Alibaba-cloud, AMD, ARM, Edgeless Systems, IBM, Intel, Microsoft, Nvidia, Red Hat, Rivos, etc.

## Motivation

At the core of a confidential computing solution lies [Trusted Execution Environments](https://en.wikipedia.org/wiki/Trusted_execution_environment) (TEEs), and it is this foundational idea that propelled the inception of the CoCo project.

> _TEEs represent isolated environments endowed with heightened security, a shield crafted by confidential computing (CC) capable hardware. This security fortress stands guard, ensuring that **applications and data remain impervious to unauthorized access or tampering during their active use**._

The driving force behind CoCo is the seamless integration of TEE infrastructure into the realm of cloud-native computing. By bridging the gap between TEEs and the cloud-native world, the project strives to bring enhanced security to the forefront of modern computing practices.

**The overarching goal of CoCo is ambitious yet clear: _standardize confidential computing at the container level and simplify its integration into Kubernetes_.**

The aim is to empower Kubernetes users to deploy confidential container workloads effortlessly, using familiar workflows and tools. CoCo envisions a future where Kubernetes users can embrace the benefits of confidential computing without the need for extensive knowledge of the underlying technologies, making security an integral and accessible aspect of their everyday operations.

## Mechanics

CoCo helps in deploying your workload that extends beyond the confines of your own infrastructure. Whether it's a cloud provider's domain, a separate division within your organization, or even an external entity, CoCo empowers you to confidently entrust your workload to diverse hands.

This capability hinges on a fundamental approach: encrypting your workload's memory and fortifying other essential low-level resources at the hardware level. This memory protection ensures that, regardless of the hosting environment, your data remains shielded, and unauthorized access is thwarted.

A key aspect of CoCo's mechanics lies in the use of cryptography-based proofs which involve employing cryptographic techniques to create verifiable evidence, such as signatures or hashes, ensuring the integrity of your software. These serve a dual purpose: validating that your software runs untampered and, conversely, preventing the execution of your workload if any unauthorized alterations are detected.

In essence, CoCo employs cryptographic mechanisms to provide assurance, creating a secure foundation that allows your software to operate with integrity across varied and potentially untrusted hosting environments.

## Foundational Principles

The project puts a strong emphasis on delivering practical cloud-native solution:

- **Simplicity:** CoCo places a premium on simplicity, employing a dedicated Kubernetes operator for deployment and configuration. This strategic choice aims to maximize accessibility by abstracting away much of the hardware-dependent intricacies, ensuring a user-friendly experience.

- **Stability:** Supporting continuous integration (CI) for the key workflows of the release.

- **Use case driven development:** CoCo adopts a use case-driven development approach, rallying the community around a select set of key use cases. Rather than a feature-centric model, this strategy ensures that development efforts are purposeful, with a spotlight on supporting essential use cases. This pragmatic approach aligns the project with real-world needs, making CoCo a solution crafted for practical cloud-native scenarios.

## Community

Discover the vibrant CoCo community and explore ways to actively engage with the project by visiting our dedicated [community page](https://confidentialcontainers.org/community/). We welcome and actively seek your thoughts, feedback, and potential contributions. Join us in shaping the future of confidential containers and explore collaborative opportunities to integrate CoCo with other cloud-native projects. Your participation is not just encouraged; it's integral to the evolution and success of this open-source initiative. Visit the community page now to be a part of the conversation and contribute to the advancement of confidential computing in the cloud-native ecosystem.

See our [CoCo community meeting notes](https://docs.google.com/document/d/1E3GLCzNgrcigUlgWAZYlgqNTdVwiMwCRTJ0QnJhLZGA/edit#heading=h.qo5uv6tg7dfy) for details on the weekly meetings, recordings, slack channels and more.
