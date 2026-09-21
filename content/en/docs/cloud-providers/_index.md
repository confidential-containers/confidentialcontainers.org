---
title: Cloud Providers
description: Cloud API adapters for Confidential Containers
categories:
  - cloud-providers
  - azure
  - google
  - aws
  - alibaba
  - gke
  - aks
weight: 53 
---

# Introduction

This guide covers how to run Confidential Containers workloads across supported cloud and virtualization environments by using the Cloud API Adaptor (CAA) and the peer-pods runtime. 
This deployment model provides strong workload isolation without tying the Kubernetes worker nodes to a particular virtualization or confidential-computing technology.
Because PodVMs are provisioned alongside the Kubernetes worker nodes rather than inside them, the worker nodes can themselves be virtual machines; bare-metal workers and nested virtualization are not required.

{{% alert title="Note" color="primary" %}}
The capabilities available to a workload depend on the infrastructure provider, region, machine type, and guest image. 
A PodVM can use a supported Trusted Execution Environment (TEE), such as Intel® TDX, AMD SEV-SNP, or IBM Secure Execution, where available. 
Some environments also support non-confidential VMs for development and testing. 
Consult the provider-specific guide for current requirements and limitations.
{{% /alert %}}

At a high level, the Cloud API Adaptor integrates with a provider's compute API or virtualization backend to provision a suitable VM for each pod.
The CAA then coordinates with the peer-pods runtime to run the pod workload in that dedicated PodVM. 
When the selected environment provides a TEE, this architecture can protect the workload from the host or hypervisor and support attestation.

Application manifests remain largely portable between environments. 
Provider-specific differences are handled during cluster setup and CAA configuration, including credentials, networking, regions or zones, instance types, VM images, and confidential-computing settings.

The provider-specific sections describe these differences and identify compatible machine types and isolation technologies where available.


## Cloud API Adaptor

The Cloud API Adaptor is an implementation of the [remote hypervisor interface](https://github.com/kata-containers/kata-containers/blob/main/src/runtime/virtcontainers/remote.go) of [Kata Containers](https://github.com/kata-containers/kata-containers).
The CAA provisions Kata Containers VMs through the configured infrastructure backend instead of starting them on the worker nodes.

The background and description of the components involved in `peer pods` can be found in the [CoCo architecture documentation](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/docs/architecture.md).


## PodVM

In Confidential Containers, a **PodVM** is the *confidential virtual machine created to host a single Kubernetes Pod*.
It is the VM boundary used by the peer-pods deployment model: when a pod is scheduled to a Confidential Containers runtime class, the runtime provisions a dedicated VM for that pod and then starts the pod’s containers **inside** the VM.

A PodVM typically includes:

- a minimal guest OS,
- a container runtime (for example, containerd), and
- an agent that coordinates container lifecycle operations inside the guest.

From Kubernetes’ point of view, you still create a normal `Pod` (or a `Deployment`, `Job`, etc.).
The PodVM is an implementation detail of the runtime that provides a stronger isolation boundary than traditional containers:

- **No shared host kernel**: the pod runs with its own guest kernel instead of the worker node’s kernel.
- **Dedicated VM boundary**: the pod’s execution is isolated from other pods by virtualization.
- **Confidential computing support**: when backed by Intel® TDX, memory inside the PodVM is protected from the host/hypervisor, and the platform can be attested before secrets are released.

In this guide, PodVMs are created on demand by the peer-pods runtime. The Cloud API Adaptor uses the configured infrastructure backend to provision and manage the VM instances that run those workloads.


## Peer Pods

**Peer Pods** is a Confidential Containers deployment model where *each Kubernetes Pod runs inside its own dedicated confidential VM* (a PodVM).

Operationally, when you deploy a pod using a Confidential Containers `RuntimeClass`, the peer-pods components:

- request the creation of a PodVM for that pod, using the configured isolation technology where available,
- boot a minimal guest environment in the PodVM, and
- start the pod’s containers inside the PodVM rather than on the Kubernetes worker node.

Across managed Kubernetes, self-managed clusters, and supported virtualization environments, the Cloud API Adaptor provides the infrastructure integration needed to provision the VM instances used for these PodVMs.

Key properties of Peer Pods:

- **Strong isolation per pod**: Each pod gets a separate VM boundary. When a supported TEE is enabled, it can also protect the pod's memory from the host or hypervisor.
- **Kubernetes-native workflow**: You deploy standard Kubernetes resources (Pods/Deployments/Jobs). The runtime handles VM lifecycle without changing application manifests.
- **Attestation-ready**: When supported by the selected TEE and environment, the PodVM can be attested to verify platform and workload identity before sensitive data is released.
- **Infrastructure-independent architecture**: Provider-specific provisioning is handled by the Cloud API Adaptor, allowing peer pods to use compatible VM types across supported environments.

The following provider-specific sections cover cluster prerequisites, enabling Confidential Containers, selecting compatible VM types and images, configuring available isolation technologies, and deploying sample workloads using peer pods.
