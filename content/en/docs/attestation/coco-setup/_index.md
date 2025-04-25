---
title: CoCo Setup
description: Setting up attestation with CoCo 
weight: 1
categories:
- attestation
tags:
- trustee
- attestation
- setup
---

If you are using Trustee with Confidential Containers, you'll need to point
your CoCo workload to your Trustee.

In your pod definition, add the following annotation.
```bash
io.katacontainers.config.hypervisor.kernel_params: "agent.aa_kbc_params=cc_kbc::http://<kbs-ip>:<kbs-port>"
```

The KBS IP will be the address of whatever system you run Trustee on in the next steps.
Make sure this is accessible within your guest. Don't use localhost.

By default the KBS port will be `8080`. You can verify this in the next steps.

A full workload definition with the annotation might look like this.
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
  annotations:
    io.containerd.cri.runtime-handler: kata-qemu-coco-dev
    io.katacontainers.config.hypervisor.kernel_params: "agent.aa_kbc_params=cc_kbc::http://<kbs-ip>:<kbs-port>"
spec:
  containers:
  - image: bitnami/nginx:1.22.0
    name: nginx
  dnsPolicy: ClusterFirst
  runtimeClassName: kata-qemu-coco-dev
```

The Trustee address can also be configured via init data.
