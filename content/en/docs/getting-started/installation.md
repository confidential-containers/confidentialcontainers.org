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

#### Create custom resource for kata

Creating a custom resource installs the required CC runtime pieces into the cluster node and creates
the `RuntimeClasses`

```
kubectl apply -k github.com/confidential-containers/operator/config/samples/ccruntime/<CCRUNTIME_OVERLAY>?ref=<RELEASE_VERSION>
```

The current present overlays are: `default` and `s390x`

For example, to deploy the `v0.10.0` release for `x86_64`, run:
```
kubectl apply -k github.com/confidential-containers/operator/config/samples/ccruntime/default?ref=v0.10.0
```

And to deploy `v0.10.0` release for `s390x`, run:
```
kubectl apply -k github.com/confidential-containers/operator/config/samples/ccruntime/s390x?ref=v0.10.0
```

Wait until each pod has the STATUS of Running.

```
kubectl get pods -n confidential-containers-system --watch
```

#### Create custom resource for enclave-cc

**Note** For `enclave-cc` certain configuration changes, such as setting the
URI of the KBS, must be made **before** applying the custom resource. 
Please refer to the [guide](./guides/enclave-cc.md#configuring-enclave-cc-custom-resource-to-use-a-different-kbc)
to modify the enclave-cc configuration.

Please see the [enclave-cc guide](./guides/enclave-cc.md) for more information.

`enclave-cc` is a form of Confidential Containers that uses process-based isolation.
`enclave-cc` can be installed with the following custom resources.
```
kubectl apply -k github.com/confidential-containers/operator/config/samples/enclave-cc/sim?ref=<RELEASE_VERSION>
```
or
```
kubectl apply -k github.com/confidential-containers/operator/config/samples/enclave-cc/hw?ref=<RELEASE_VERSION>
```
for the **simulated** SGX mode build or **hardware** SGX mode build, respectively.

### Verify Installation

Check the `RuntimeClasses` that got created.

```
kubectl get runtimeclass
```
Output:
```
NAME                 HANDLER              AGE
kata                 kata-qemu            8d
kata-clh             kata-clh             8d
kata-qemu            kata-qemu            8d
kata-qemu-coco-dev   kata-qemu-coco-dev   8d
kata-qemu-sev        kata-qemu-sev        8d
kata-qemu-snp        kata-qemu-snp        8d
kata-qemu-tdx        kata-qemu-tdx        8d
```

Details on each of the runtime classes:

- `kata` - Convenience runtime that uses the handler of the default runtime
- `kata-clh` - standard kata runtime using the cloud hypervisor
- `kata-qemu` - same as kata
- `kata-qemu-coco-dev` - standard kata runtime using the QEMU hypervisor including all CoCo building blocks for a non CC HW
- `kata-qemu-sev` - using QEMU, and support for AMD SEV HW
- `kata-qemu-snp` - using QEMU, and support for AMD SNP HW
- `kata-qemu-tdx` - using QEMU, and support Intel TDX HW based on what's provided by [Ubuntu](https://github.com/canonical/tdx) and [CentOS 9 Stream](https://sigs.centos.org/virt/tdx/).


If you are using `enclave-cc` you should see the following runtime classes.

```
kubectl get runtimeclass
```
Output:
```
NAME            HANDLER         AGE
enclave-cc      enclave-cc      9m55s
```
