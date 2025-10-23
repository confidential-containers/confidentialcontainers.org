---
title: Confidential AI
description: Confidential Containers for AI 
categories:
- use cases
tags:
- AI 
- ML
weight: 60
---

# Accelerated Inference and Fine Tuning

Many AI workloads require or benefit from accelerators like GPUs. CoCo supports Confidential
Computing compliant accelerators.

Workloads request compliant accelerators using standard Kubernetes resource descriptions in the pod
specification.

CoCo includes a specific feature called, “composite attestations” to bind evidence of accelerators
and vCPUs. Users can enable policies requiring composite attestations to gate releasing secret
resources to the pod.


# Retrieval Augmented Generation

In addition to the general AI usages mentioned above, CoCo can help secure popular RAG deployments.
RAG deployments often include repositories of confidential information used to augment the LLM’s
latent knowledge. To secure those sensitive repositories including their representations in vector
databases, deploy all containers using CoCo.

Access to the confidential repositories and vector databases should be gated on correct attestation
verification. Attestation policies can be managed using Trustee or similar attestation services.

CoCo includes features in “Confidential Data Hub” to securely handle secrets for resources like
encrypted storage that can be used for confidential document repositories. See the Features section
for Secret Resources and Sealed Secrets.


# Federated Learning

Federated Learning (FL) is a decentralized machine learning approach where multiple participants
(such as organizations, edge devices, or distributed servers) collaboratively train a model without
sharing their raw data. Instead of centralizing data, each participant trains a local model and
shares only model updates with a central aggregator, preserving data privacy and reducing
communication overhead.  A variety of FL systems and frameworks currently exist and can be deployed
more securely using CoCo.

## CoCo's Role and Guidance

FL requires honest participants and a trusted aggregator. CoCo can help enforce honest behavior of
each client, reducing the risk of poisoning and tampering at each client. CoCo can also protect the
aggregator from infrastructure threats and malicious clients.

To secure an FL workload with CoCo:
* Deploy the Aggregator in a Confidential Container to protect the central model aggregation service
  and the global model weights from tamper or unauthorized exposure.
* Deploy each client in a Confidential Container to protect local data and model weights from
  external attack as well as reducing the risk of poisoning and tampering.
* Mutual Trust:
* The Aggregator must only accept updates from correctly attesting clients. The attestation
  measurements of the container ensure that only the expected training code is calculating and
  providing model updates.
* Each Client should submit local model weights to or accept global model weights from a correctly
  attesting Aggregator.
    * In lieu of incorporating verification logic in each component, deployments can use attestation
      policies in Trustee. Clients and Aggregators can receive communication credentials (e.g. TLS
      Certificates, JWTs) from Trustee upon successful attestation verification.

