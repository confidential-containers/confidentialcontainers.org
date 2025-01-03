---
title: Get Attestation
date: 2025-01-08
description: Workloads that request attestation evidence
categories:
- feature 
tags:
- attestation 
---

{{% alert title="Warning" color="primary" %}}
This feature is disabled by default due to some subtle security considerations
which are described below.
{{% /alert %}}

Workloads can directly request attestation evidence.
A workload could use this evidence to carry out its own attestation protocol.

### Enabling

To enable this feature, set the following parameter in the guest kernel command line.
```bash
agent.guest_components_rest_api=all
```

As usual, command line configurations can be added with annotations.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        io.katacontainers.config.hypervisor.kernel_params: "agent.guest_components_rest_api=all"
    spec:
      runtimeClassName: (...)
      containers:
      - name: nginx
        (...)
```
### Attestation

Once enabled, an attestation can be retrieved via the REST API.
```bash
curl http://127.0.0.1:8006/aa/evidence?runtime_data=xxxx
```

The API expects runtime data to be provided. The runtime data will be included in the report. 
Typically this is used to bind a nonce or the hash of a key to the evidence.

### Security

Enabling this feature can make your workload vulnerable to so-called "evidence factory" attacks.
By default, different CoCo workloads can have the same TCB, even if they use different container images.
This is a design feature, but it means that when the attestation report API is enabled,
two workloads could produce interchangeable attestation reports, even if they are operated by different parties.

To make sure someone else cannot generate evidence that could be used with your attestation protocol,
configure your guest to differentiate its TCB.
For example, use the init-data (which is measured) to specify the public key of the KBS.
Init-data is currently only supported with peer pods.
More information about it will be added soon.

