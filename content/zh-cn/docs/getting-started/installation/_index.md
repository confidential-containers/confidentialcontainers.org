---
title: 安装
description: 使用 Helm Chart 安装 Confidential Containers
weight: 11
categories:
  - getting-started
tags:
  - helm
  - installation
---

{{% alert title="说明" color="primary" %}}
安装 Confidential Containers 之前，请先完成[前提条件](../prerequisites/)章节中的准备工作。
{{% /alert %}}

## 使用 Helm 安装 CoCo

通过 Confidential Containers Charts 仓库提供的 Helm Chart 安装 CoCo 运行时。

{{< tabpane text=true right=true >}}
{{% tab header="最新版本" lang="bash" %}}
安装最新版本：
```bash
helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  --namespace coco-system \
  --create-namespace
```
{{% /tab %}}
{{% tab header="指定版本" lang="bash" %}}
将 `<VERSION>` 替换为所需的[发布版本](https://github.com/confidential-containers/charts/releases)：

```bash
helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  --version <VERSION> \
  --namespace coco-system \
  --create-namespace
```

例如，安装 `v0.18.0` 版本：

```bash
helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  --version 0.18.0 \
  --namespace coco-system \
  --create-namespace
```
{{% /tab %}}
{{< /tabpane >}}

等待所有 Pod 的 `STATUS` 变为 `Running`。

```bash
kubectl get pods -n coco-system --watch
```

如果需要了解特定场景的安装选项（例如 `s390x`、`peer-pods`）或高级配置，请参见
[Charts 仓库文档](https://github.com/confidential-containers/charts)。

### 验证安装

检查预期的 RuntimeClass 是否已创建。

```bash
kubectl get runtimeclass
```

可用的 RuntimeClass 取决于具体架构：

{{< tabpane text=true right=true >}}
{{% tab header="x86_64" lang="bash" %}}
| runtimeclass | 说明 |
| ------------ | ---- |
| `kata-qemu-coco-dev` | 开发/测试运行时 |
| `kata-qemu-coco-dev-runtime-rs` | 开发/测试运行时（基于 Rust） |
| `kata-qemu-snp` | AMD SEV-SNP |
| `kata-qemu-tdx` | Intel TDX |
| `kata-qemu-nvidia-gpu-snp` | NVIDIA GPU 加 AMD SNP 组合 |
| `kata-qemu-nvidia-gpu-tdx` | NVIDIA GPU 加 Intel TDX 组合|
{{% /tab %}}
{{% tab header="s390x" lang="bash" %}}
| runtimeclass | 说明 |
| ------------ | ---- |
| `kata-qemu-coco-dev` | 开发/测试运行时 |
| `kata-qemu-coco-dev-runtime-rs` | 开发/测试运行时（基于 Rust） |
| `kata-qemu-se` | IBM Secure Execution |
| `kata-qemu-se-runtime-rs` | IBM Secure Execution（基于 Rust） |
{{% /tab %}}
{{% tab header="peer-pods" lang="bash" %}}
| runtimeclass | 说明 |
| ------------ | ---- |
| `kata-remote` | Peer-pods |
{{% /tab %}}
{{< /tabpane >}}

### 卸载

如需卸载 Confidential Containers 并删除 `coco-system` 命名空间，请执行：

```bash
helm uninstall coco --namespace coco-system
kubectl delete namespace coco-system
```