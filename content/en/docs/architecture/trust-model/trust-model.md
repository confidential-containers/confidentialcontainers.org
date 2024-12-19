---
title: Trust Model for Confidential Containers
date: 2024-01-23
description: Overview of Confidential Containers security
categories:
- docs
tags:
- docs
- trust-model
weight: 1
---

Confidential Containers mainly relies on VM enclaves, where the guest does not trust the host.
Confidential computing, and by extension Confidential Containers, provides technical assurances
that the untrusted host cannot access guest data or manipulate guest control flow.

### Trusted 

Confidential Containers maps pods to confidential VMs, meaning that everything inside a pod is
within an enclave. In addition to the workload pod, the guest also contains helper processes
and daemons to setup and control the pod.
These include the `kata-agent` and guest components as described in the architecture section.

More specifically, the guest is defined as four components.
- Guest firmware
- Guest kernel
- Guest kernel command line
- Guest root filesystem

All platforms supported by Confidential Containers must measure these four components.
Details about the mechanisms for each platform are below.

Note that the hardware measurement usually does not directly cover the workload containers.
Instead, containers are covered by a second-stage of measurement that uses generic OCI
standards such as signing.
This second stage of measurement is rooted in the trust of the first stage,
but decoupled from the guest image.

Confidential Containers also relies on an external trusted entity, usually Trustee,
to attest the guest.

### Untrusted

Everything on the host outside of the enclave is untrusted.
This includes the Kubelet, CRI runtimes like containerd, the host kernel,
the Kata Shim, and more.

Since the Kubernetes control plane is untrusted, some traditional Kubernetes
security techniques are not relevant to Confidential Containers without special considerations.

### Crossing the trust boundary

In confidential computing careful scrutiny is required whenever information crosses the boundary
between the trusted and untrusted contexts. 
Secrets should not leave the enclave without protection
and entities outside of the enclave should not be able to trigger malicious behavior inside the guest. 

In Confidential Containers there are APIs that cross the trust boundary.
The main example is the API between the Kata Agent in the guest and the Kata Shim on the host.
This API is protected with an OPA policy running inside the guest that can block
malicious requests by the host.

Note that the kernel command line, which is used to configure the Kata Agent, does not
cross the trust boundary because it is measured at boot.
Assuming that the guest measurement is validated, the APIs that are most significant
are those that are not measured by the hardware.

Quantifying the attack surface of an API is non-trivial.
The Kata Agent can perform complex operations such as mounting a block device provided
by the host.
In the case that a host-provided device is attached to the guest the attack surface
is extended to any information provided by this device.
It's also possible that any of the code used to implement the API inside the guest
has a bug in it.
As the complexity of the API increases, the likelihood of a bug increases.
The nuances of the Kata Agent API is why Confidential Containers relies on a dynamic
and user-configurable policy to either block endpoints entirely
or only allow particular types of requests to be made.
For example, the policy can be used to make sure that a block device is mounted
only to a particular location.

Applications deployed with Confidential Containers should also be aware of the trust boundary.
An application running inside of an enclave is not secure if it exposes a dangerous API to the outside world.
Confidential applications should almost always be deployed with signed and/or encrypted images.
Otherwise the container image itself can be considered as part of the unmeasured API.

### Out of Scope

Some attack vectors are out of scope of confidential computing and Confidential Containers.
For instance, confidential computing platforms usually do not protect against hardware side-channels.
Neither does Confidential Containers.
Different hardware platforms and platform generations may have different guarantees
regarding properties like memory integrity.
Confidential Containers inherits the properties of whatever TEE it is using.

Confidential computing does not protect against denial of service.
Since the untrusted host is in charge of scheduling, it can simply not run the guest.
This is true for Confidential Containers as well.
In Confidential Containers the untrusted host can avoid scheduling the pod VM
and the untrusted control plane can avoid scheduling the pod.
These are seen as equivalent.

In general orchestration is untrusted in Confidential Containers.
Confidential Containers provides few guarantees about where, when, or in what order
workloads run, besides that the workload is deployed inside of a genuine enclave
containing the expected software stack.

### Cloud Native Personas

So far the trust model has been described in terms of a host and a guest,
following from the underlying confidential computing trust model,
but these terms are not used in cloud native computing. 
How do we understand the trust model in terms of cloud native personas?
Confidential Containers is a flexible project.
It does not explicitly define how parties should interact. 
but some possible arrangements are described in the [personas section](cloud-native-personas.md).

### Measurement Details

As mentioned above, all hardware platforms must measure the four components representing
the guest image.
This table describes how each platform does this.

| Platform | Firmware | Kernel | Command Line | Rootfs |
| -------- | -------- | ------ | ------------ | ------ |
| SEV-SNP  | Pre-measured by ASP | Measured direct boot via OVMF | Measured direct boot | Measured direct boot |
| TDX | Pre-launch measurement | RTMR | RTMR | Dm-verity hash provided in command line |
| SE | Included in encrypted SE image | included in SE image | included in SE image | included in SE image |

### See Also

- Confidential Computing Consortium (CCC) published
  "[A Technical Analysis of Confidential Computing](https://confidentialcomputing.io/wp-content/uploads/sites/10/2023/03/CCC-A-Technical-Analysis-of-Confidential-Computing-v1.3_unlocked.pdf)"
  section 5 of which defines the threat model for confidential computing.
- CNCF Security Technical Advisory Group published
  "[Cloud Native Security Whitepaper](https://github.com/cncf/tag-security/blob/main/security-whitepaper/v2/CNCF_cloud-native-security-whitepaper-May2022-v2.pdf)"
- Kubernetes provides documentation :
  "[Overview of Cloud Native Security](https://kubernetes.io/docs/concepts/security/overview/)"
- Open Web Application Security Project -
  "[Docker Security Threat Modeling](https://github.com/OWASP/Docker-Security/blob/main/001%20-%20Threats.md)"
