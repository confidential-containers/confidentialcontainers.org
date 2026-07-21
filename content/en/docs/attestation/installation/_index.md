---
title: Installation 
description: Installing Trustee 
weight: 10
categories:
- attestation
tags:
- trustee
- attestation
- installation
---

Trustee can be deployed in several different configurations.
In every installation scenario, deploy Trustee in a trusted environment.
This could be a local server, some trusted third party, or even another enclave.
Official support for deploying Trustee inside of Confidential Containers
is being developed.

## Choosing an installation method

| Method | Best for |
|--------|----------|
| [Helm](helm/) | Development or production deployments, where you want declarative configuration or advanced options (custom storage, BYOK, IBM SE) |
| [Trustee Operator](kubernetes/) | Kubernetes deployments with operator-managed lifecycle  |
| [Docker Compose](docker/) | Local evaluation, development, and testing |



