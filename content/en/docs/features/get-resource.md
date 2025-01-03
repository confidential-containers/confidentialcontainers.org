---
title: Get Secret Resources 
date: 2025-01-08
description: Workloads that request resources from Trustee 
categories:
- feature 
tags:
- resources 
---

{{% alert title="Note" color="primary" %}}
Requesting secret resources depends on attestation.
[Configure attestation](../attestation) before requesting resources
{{% /alert %}}

While [sealed secrets](../sealed-secrets) can be used to store encrypted secret resources
as Kubernetes Secrets and have them transparently decrypted using keys from Trustee,
a workload can also explicitly request resources via a REST API
that is automatically exposed to the pod network namespace.

For example, you can run this command from your container.
```bash
curl http://127.0.0.1:8006/cdh/resource/default/key/1
```

In this example Trustee will fulfill the request for `kbs:///default/key/1`
assuming that the resource has been provisioned.
