---
title: What is Attestation?

description: Background on attestation 
weight: 1
categories:
- attestation
tags:
- trustee
- attestation
---

## Overview

Generally speaking, confidential computing has two dimensions.
First, there are runtime isolation technologies such as memory encryption.
These create a boundary between the TEE and the untrusted world.
This boundary protects data while it is in use inside the TEE.

Runtime isolation does not account for the initial state of the TEE,
which is typically setup by an untrusted host.
For example, the hypervisor usually sets the initial memory of a guest.
Since the configuration and initial state of a TEE is highly significant,
a TEE should not be considered secure unless these properties are validated. 

Attestation, the second aspect of confidential computing, is the process
of establishing trust in a TEE.
More precisely, attestation extends trust from a hardware root of trust, to a TEE.
The hardware root of trust attests to the configuration and initial state of the TEE,
typically by signing a report about the TEE with a key linked to the hardware manufacturer.

Attestation goes beyond primitives such as measurement and report signing
to encompass a process for establishing trust between parties. 
Often, attestation is described from the perspective of the TEE,
but this can lead to confusion.

In its simplest form, attestation involves two parties; the hardware manufacturer
and the owner of a workload that will be deployed in a TEE.
These parties already trust each other.
The goal is for the workload owner to trust a TEE environment setup by a mostly untrusted host.
Most people describe this as a guest attesting to a workload owner or a relying party.
This is a reasonable simplification where the hardware evidence represents
the hardware manufacturer more generally.

Trustee conceptualizes this process in terms of resource requests.
When Trustee receives a request for a resource, Trustee determines whether
the requester is trustworthy before granting access to the resource.

## Disambiguation

Unfortunately, the term attestation is slightly overloaded.
Attestation can also refer to host attestation, which is often based on TPMs.
While confidential attestation can involve vTPMs,
confidential attestation is scoped to a TEE
and usually ignores the untrusted host.

Host attestation usually has a continuous and non-blocking model 
where hosts are periodically attested.
If attestation fails, some corrective action, such as quarantining the node,
can be taken.

Confidential attestation, on the other hand, usually blocks the delivery of secrets
to a TEE, and it is often scoped to the initial state of a guest.

There is overlap between these two approaches,
but crucially the party that can validate a host environment,
and the party that can validate a guest environment
are usually not the same. 
Thus, there are two distinct processes.

SPIFFE/SPIRE also uses the term attestation.
With SPIRE, node attestation is somewhat similar to host attestation,
but workload attestation refers more generally to one component
attesting to the properties of another.

## Workload Design

Trustee and the KBS Protocol do not specify how or when a workload
should invoke the attestation process.
Of course, attestation must take place to get access to a resource from Trustee.
With Confidential Containers, attestation happens lazily; only when the first
operation that depends on a resource takes place.
This could happen prior to container startup to get access to an image decryption key
or a sealed secret, or it could happen later.
In Confidential Containers, if a workload does not depend on any resources, it won't be attested.
For various reasons, this would not be a secure workload.
On the other hand, if a workload does depend on a resource, and it receives that resource
it is implied that attestation has taken place, particularly if the resource
is a secret known only to Trustee.
