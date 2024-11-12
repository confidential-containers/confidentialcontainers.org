---
title: Installation 
description: Installing Confidential Containers with the operator 
weight: 20
categories:
- getting-started 
tags:
- operator
- installation
---
{{% alert title="Note" color="primary" %}}
Make sure you have completed the pre-requisites before installing Confidential Containers.
{{% /alert %}}

### Deploy the operator

Deploy the operator by running the following command  where `<RELEASE_VERSION>` needs to be substituted
with the desired [release tag](https://github.com/confidential-containers/operator/tags).

```
kubectl apply -k github.com/confidential-containers/operator/config/release?ref=<RELEASE_VERSION>
```

For example, to deploy the `v0.10.0` release run:
```
kubectl apply -k github.com/confidential-containers/operator/config/release?ref=v0.10.0
```

Wait until each pod has the STATUS of Running.

```
kubectl get pods -n confidential-containers-system --watch
```

### Create the custom resource

Creating a custom resource installs the required CC runtime pieces into the cluster node and creates
the runtime classes. 

{{< tabpane text=true right=true >}}
 {{% tab header="x86" lang="bash" %}}
	kubectl apply -k github.com/confidential-containers/operator/config/samples/ccruntime/default?ref=<RELEASE_VERSION>
  {{% /tab %}}
  {{% tab header="s390x" lang="bash" %}}
	kubectl apply -k github.com/confidential-containers/operator/config/samples/ccruntime/s390x?ref=<RELEASE_VERSION>
  {{% /tab %}}
  {{% tab header="SGX" lang="bash" %}}
	kubectl apply -k github.com/confidential-containers/operator/config/samples/enclave-cc/hw?ref=<RELEASE_VERSION>
  {{% /tab %}}
{{< /tabpane >}}

{{% alert title="Note" color="primary" %}}
If using enclave-cc with SGX, please refer to [this guide](./guides/enclave-cc.md#configuring-enclave-cc-custom-resource-to-use-a-different-kbc)
for more information on setting the custom resource.
{{% /alert %}}


Wait until each pod has the STATUS of Running.

```
kubectl get pods -n confidential-containers-system --watch
```

### Verify Installation

See if the expected runtime classes were created.
```
kubectl get runtimeclass
```
Should return

{{< tabpane text=true right=true >}}
 {{% tab header="x86" lang="bash" %}}
	NAME                 HANDLER              AGE
	kata                 kata-qemu            8d
	kata-clh             kata-clh             8d
	kata-qemu            kata-qemu            8d
	kata-qemu-coco-dev   kata-qemu-coco-dev   8d
	kata-qemu-sev        kata-qemu-sev        8d
	kata-qemu-snp        kata-qemu-snp        8d
	kata-qemu-tdx        kata-qemu-tdx        8d
  {{% /tab %}}
  {{% tab header="s390x" lang="bash" %}}
	NAME           HANDLER        AGE
	kata           kata-qemu      60s
	kata-qemu      kata-qemu      61s
	kata-qemu-se   kata-qemu-se   61s
  {{% /tab %}}
  {{% tab header="SGX" lang="bash" %}}
	NAME            HANDLER         AGE
	enclave-cc      enclave-cc      9m55s
  {{% /tab %}}
{{< /tabpane >}}

#### Runtime Classes

CoCo supports many different runtime classes.
Different deployment types install different sets of runtime classes.
The operator may install some runtime classes that are not valid for your system.
For example, if you run the operator on a TDX machine, you might have TDX and SEV
runtime classes. Use the runtime classes that match your hardware.

| Name | Type | Description |
| ---- | ---- | ----------- |
| `kata` | x86  | Alias of the default runtime handler (usually the same as `kata-qemu`) |
| `kata-clh` | x86 | Kata Containers (non-confidential) using Cloud Hypervisor |
| `kata-qemu` | x86 | Kata Containers (non-confidential) using QEMU |
| `kata-qemu-coco-dev` | x86 | CoCo without an enclave (for testing only) |
| `kata-qemu-sev` | x86 | CoCo with QEMU for AMD SEV HW |
| `kata-qemu-snp` | x86 | CoCo with QEMU for AMD SNP HW |
| `kata-qemu-tdx` | x86 | CoCo with QEMU for Intel TDX HW |
| `kata-qemu-se` | s390x | CoCO with QEMU for Secure Execution |
| `enclave-cc` | SGX | CoCo with enclave-cc (process-based isolation without Kata) |
