---
title: Installation
description: Installing Confidential Containers with Helm charts
weight: 11
categories:
  - getting-started
tags:
  - helm
  - installation
---

{{% alert title="Note" color="primary" %}}
Make sure you have completed the pre-requisites before installing Confidential Containers.
{{% /alert %}}

## Install CoCo with Helm

Install the CoCo runtime using the Helm chart from the Confidential Containers charts
repository.

{{< tabpane text=true right=true >}}
{{% tab header="Latest release" lang="bash" %}}
Install the latest released version:
```bash
helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  --namespace coco-system \
  --create-namespace
```
{{% /tab %}}
{{% tab header="Pinned version" lang="bash" %}}
Substitute `<VERSION>` with the desired [release version](https://github.com/confidential-containers/charts/releases):

```bash
helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  --version <VERSION> \
  --namespace coco-system \
  --create-namespace
```

For example, to install version v0.18.0:

```bash
helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  --version 0.18.0 \
  --namespace coco-system \
  --create-namespace
```
{{% /tab %}}
{{< /tabpane >}}

Wait until each pod has the STATUS of Running.

```bash
kubectl get pods -n coco-system --watch
```

For platform-specific installation options (s390x, peer-pods, etc.) and advanced configuration,
see the [charts repository documentation](https://github.com/confidential-containers/charts).

### Verify Installation

See if the expected runtime classes were created.
```bash
kubectl get runtimeclass
```

The available runtimeclasses depend on the architecture:

{{< tabpane text=true right=true >}}
{{% tab header="x86_64" lang="bash" %}}
| runtimeclass | Description |
| ------------ | ----------- |
| `kata-qemu-coco-dev` | Development/testing runtime |
| `kata-qemu-coco-dev-runtime-rs` | Development/testing runtime (Rust-based) |
| `kata-qemu-snp` | AMD SEV-SNP |
| `kata-qemu-tdx` | Intel TDX |
| `kata-qemu-nvidia-gpu-snp` | NVIDIA GPU with AMD SEV-SNP protection |
| `kata-qemu-nvidia-gpu-tdx` | NVIDIA GPU with Intel TDX protection |
{{% /tab %}}
{{% tab header="s390x" lang="bash" %}}
| runtimeclass | Description |
| ------------ | ----------- |
| `kata-qemu-coco-dev` | Development/testing runtime |
| `kata-qemu-coco-dev-runtime-rs` | Development/testing runtime (Rust-based) |
| `kata-qemu-se` | IBM Secure Execution |
| `kata-qemu-se-runtime-rs` | IBM Secure Execution (Rust-based) |
{{% /tab %}}
{{% tab header="peer-pods" lang="bash" %}}
| runtimeclass | Description |
| ------------ | ----------- |
| `kata-remote` | Peer-pods |
{{% /tab %}}
{{< /tabpane >}}

### Uninstall

To uninstall Confidential Containers and delete the `coco-system` namespace, run:
```bash
helm uninstall coco --namespace coco-system
kubectl delete namespace coco-system
```