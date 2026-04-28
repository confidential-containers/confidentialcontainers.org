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
You can do this with an annotation on your workload, or via init-data.

## Annotation

In your pod definition, add the following annotation.
```yaml
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

## Init-Data

The KBS URI can be set via Init-Data.
Add the KBS URI to both the Attestation Agent config file (`aa.toml`)
and to the CDH config file (`cdh.toml`).
In most cases the CDH and the AA should use the same KBS.

```toml
version = "0.1.0"
algorithm = "sha384"

[data]

"aa.toml" = '''
[token_configs.kbs]
url = "http://<kbs-ip>:<kbs-port>"
'''

"cdh.toml" = '''
[kbc]
name = "cc_kbc"
url = "http://<kbs-ip>:<kbs-port>"
'''
```

See [Init-Data](../features/initdata) page for instructions on how to attach
the Init-Data to a workload.
