---
date: 2024-12-03
title: Confidential Containers without confidential hardware
linkTitle: Confidential Containers without confidential hardware
description: >
  How to use Confidential Containers without confidential hardware
author: Steve Horsman ([@stevenhorsman](https://github.com/stevenhorsman)) and Wainer dos Santos Moschetta ([@wainersm](https://github.com/wainersm))
---

> Note
This blog post was originally published here based on the very first versions of Confidential Containers (CoCo) which at that time was just a Proof-of-Concept (PoC) project. Since then the project evolved a lot: we managed to merge the work to the Kata Containers mainline, removed a code branch of Containerd, many new features were introduced/improved, new sub-projects emerged and the community finally reached its maturity. Thus, this new version of that blog post revisits the installation and use of CoCo on workstations without confidential hardware, taking into consideration the changes since the early versions of the project.

## Introduction

The [Confidential Containers](https://github.com/confidential-containers) (CoCo) project aims to implement a cloud-native solution for confidential computing using the most advanced [trusted execution environments](https://en.wikipedia.org/wiki/Trusted_execution_environment) (TEE) technologies available from hardware vendors like AMD, IBM and Intel.

The community recognizes that not every developer has access to TEE-capable machines and we don't want this to be a blocker for contributions. So version 0.10.0 and later come with a custom runtime that lets developers play with CoCo on either a simple virtual or bare-metal machine.

In this tutorial you will learn:

* How to install CoCo and create a simple confidential pod on Kubernetes
* The main features that keep your pod confidential

Since we will be using a custom runtime environment without confidential hardware,  we will not be able to do real attestation implemented by CoCo, but instead will use a sample verifier, so the pod created won't be  strictly “confidential”.

## A brief introduction to Confidential Containers

Confidential Containers is a sandbox project of the [Cloud Native Computing Foundation](https://www.cncf.io) (CNCF) that enables
cloud-native [confidential computing](https://confidentialcomputing.io/faq/) by taking advantage of a variety of hardware platforms and technologies, such as Intel SGX,
Intel TDX, AMD SEV-SNP and IBM Secure Execution for Linux. The project aims to integrate hardware and software technologies to
deliver a seamless experience to users running applications on Kubernetes.

For a high level overview of the CoCo project, please see: [What is the Confidential Containers project](https://www.redhat.com/en/blog/what-confidential-containers-project)?

## What is required for this tutorial?

As mentioned above, you don’t need TEE-capable hardware for this tutorial. You will only be required to have:

* Ubuntu 22.04 virtual or bare-metal machine with a minimum of 8GB RAM and 4 vcpus
* Kubernetes 1.30.1 or above

It is beyond the scope of this blog to tell you how to install Kubernetes, but there are some details that should be taken into consideration:

1. [CoCo v0.10.0 was tested on Continuous Integration](https://github.com/confidential-containers/confidential-containers/blob/main/releases/v0.10.0.md#hardware-support) (CI) with Kubernetes installed via kubeadm (see here how to create a cluster with that tool). Also, some community members reported that it works fine in Kubernetes over [kind](https://kind.sigs.k8s.io).
1. [Containerd](https://containerd.io) is supported and [CRI-O](https://cri-o.io) still lack some features (e.g. encrypted images). On CI, most of the tests were executed on Kubernetes configured with Containerd, so this is the chosen container runtime for this blog.
1. Ensure that your cluster nodes are not tainted with `NoSchedule`, otherwise the installation will fail. This is very common on single-node Kubernetes installed with [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm).
1. Ensure that the worker nodes where CoCo will be installed have SELinux disabled as this is a current limitation (refer to the [v0.10.0 limitations](https://github.com/confidential-containers/confidential-containers/blob/main/releases/v0.10.0.md#limitations) for further details).

## How to install Confidential Containers

The CoCo runtime is bundled in a [Kubernetes operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) that should be deployed on your cluster.

In this section you will learn how to get the [CoCo operator](https://github.com/confidential-containers/operator) installed.

First, you should have the `node.kubernetes.io/worker=` label on all the cluster nodes that you want the runtime installed on. This is how the cluster admin instructs the operator controller about what nodes, in a multi-node cluster, need the runtime. Use the command `kubectl label node NODE_NAME "node.kubernetes.io/worker="` as on the listing below to add the label:

```shell
$ kubectl get nodes
NAME    	STATUS   ROLES       	AGE   VERSION
coco-demo   Ready	control-plane   87s   v1.30.1
$ kubectl label node "coco-demo" "node.kubernetes.io/worker="
node/coco-demo labeled
```

Once the target worker nodes are properly labeled, the next step is to install the operator controller. You should first ensure that SELinux is disabled or in permissive mode, however, because the operator controller will attempt to restart services in your system and SELinux may deny that. Using the following sequence of commands we set SELinux to permissive and install the operator controller:

```shell
$ sudo setenforce 0
$ kubectl apply -k github.com/confidential-containers/operator/config/release?ref=v0.10.0
```

This will create a series of resources in the `confidential-containers-system`
namespace. In particular, it creates a deployment with pods that all need to be running before you continue the installation, as shown below:

```shell
$ kubectl get pods -n confidential-containers-system
NAME                                          	READY   STATUS	RESTARTS   AGE
cc-operator-controller-manager-557b5cbdc5-q7wk7   2/2 	Running   0      	2m42s
```

The operator controller is capable of managing the installation of different [CoCo runtimes](https://github.com/confidential-containers/operator/blob/v0.10.0/api/v1beta1/ccruntime_types.go) through [Kubernetes custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/). In v0.10.0 release, the following runtimes are supported:

* **ccruntime** - the default, Kata Containers based implementation of CoCo. **This is the runtime that we will use here**.
* **enclave-cc** - provides process-based isolation using Intel SGX

Now it is time to install the ccruntime runtime. You should run the following commands and wait a few minutes while it downloads and installs Kata Containers and configures your node for CoCo:

```shell
$ kubectl apply -k github.com/confidential-containers/operator/config/samples/ccruntime/default?ref=v0.10.0
ccruntime.confidentialcontainers.org/ccruntime-sample created
$ kubectl get pods -n confidential-containers-system --watch
NAME                                          	READY   STATUS	RESTARTS   AGE
cc-operator-controller-manager-557b5cbdc5-q7wk7   2/2 	Running   0      	26m
cc-operator-daemon-install-q27qz              	1/1 	Running   0      	8m10s
cc-operator-pre-install-daemon-d55v2          	1/1 	Running   0      	8m35s
```

You can notice that it will get installed a couple of [Kubernetes runtimeclasses](https://kubernetes.io/docs/concepts/containers/runtime-class/) as shown on the listing below. Each class defines a container runtime configuration as, for example, **kata-qemu-tdx** should be used to launch QEMU/KVM for Intel TDX hardware (similarly **kata-qemu-snp** for AMD SEV-SNP). For the purpose of creating a confidential pod in a non-TEE environment we will be using the **kata-qemu-coco-dev** runtime class.

```shell
$ kubectl get runtimeclasses
NAME             	HANDLER          	AGE
kata             	kata-qemu        	26m
kata-clh         	kata-clh         	26m
kata-qemu        	kata-qemu        	26m
kata-qemu-coco-dev    kata-qemu-coco-dev    26m
kata-qemu-sev    	kata-qemu-sev    	26m
kata-qemu-snp    	kata-qemu-snp    	26m
kata-qemu-tdx    	kata-qemu-tdx    	26m
```

## Creating your first confidential pod

In this section we will create the bare-minimum confidential pod using a regular busybox image. Later on we will show how to use encrypted container images.

You should create the **coco-demo-01.yaml** file with the content:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: coco-demo-01
  annotations:
    "io.containerd.cri.runtime-handler": "kata-qemu-coco-dev"
spec:
  runtimeClassName: kata-qemu-coco-dev
  containers:
    - name: busybox
      image: quay.io/prometheus/busybox:latest
      imagePullPolicy: Always
      command:
        - sleep
        - "infinity"
  restartPolicy: Never
```

Then you should apply that manifest and wait for the pod to be `RUNNING` as shown below:

```shell
$ kubectl apply -f coco-demo-01.yaml
pod/coco-demo-01 created
$ kubectl get pods
NAME       	READY   STATUS	RESTARTS   AGE
coco-demo-01   1/1 	Running   0      	24s
```

Congrats! Your first Confidential Containers pod has been created and you don't need confidential hardware!

## A view of what's going on behind the scenes

In this section we'll show you some concepts and details of the CoCo implementation that can be demonstrated with this simple coco-demo-01 pod. Later we should be creating more complex and interesting examples.

### Containers inside a confidential virtual machine (CVM)

Our confidential containers implementation is built on [Kata Containers](https://katacontainers.io/), whose most notable feature is running the containers in a virtual machine (VM), so the created demo pod is naturally isolated from the host kernel.

Currently CoCo supports launching pods with [QEMU](https://www.qemu.org/) only, despite Kata Containers supporting other hypervisors. An instance of QEMU was launched to run the coco-demo-01, as you can see below:

```shell
$ ps aux | grep /opt/kata/bin/qemu-system-x86_64
root   	15892  0.8  3.6 2648004 295424 ?  	Sl   20:36   0:04 /opt/kata/bin/qemu-system-x86_64 -name sandbox-baabb31ff0c798a31bca7373f2abdbf2936375a5729a3599799c0a225f3b9612 -uuid e8a3fb26-eafa-4d6b-b74e-93d0314b6e35 -machine q35,accel=kvm,nvdimm=on -cpu host,pmu=off -qmp unix:fd=3,server=on,wait=off -m 2048M,slots=10,maxmem=8961M -device pci-bridge,bus=pcie.0,id=pci-bridge-0,chassis_nr=1,shpc=off,addr=2,io-reserve=4k,mem-reserve=1m,pref64-reserve=1m -device virtio-serial-pci,disable-modern=true,id=serial0 -device virtconsole,chardev=charconsole0,id=console0 -chardev socket,id=charconsole0,path=/run/vc/vm/baabb31ff0c798a31bca7373f2abdbf2936375a5729a3599799c0a225f3b9612/console.sock,server=on,wait=off -device nvdimm,id=nv0,memdev=mem0,unarmed=on -object memory-backend-file,id=mem0,mem-path=/opt/kata/share/kata-containers/kata-ubuntu-latest-confidential.image,size=268435456,readonly=on -device virtio-scsi-pci,id=scsi0,disable-modern=true -object rng-random,id=rng0,filename=/dev/urandom -device virtio-rng-pci,rng=rng0 -device vhost-vsock-pci,disable-modern=true,vhostfd=4,id=vsock-1515224306,guest-cid=1515224306 -netdev tap,id=network-0,vhost=on,vhostfds=5,fds=6 -device driver=virtio-net-pci,netdev=network-0,mac=6a:e6:eb:34:52:32,disable-modern=true,mq=on,vectors=4 -rtc base=utc,driftfix=slew,clock=host -global kvm-pit.lost_tick_policy=discard -vga none -no-user-config -nodefaults -nographic --no-reboot -object memory-backend-ram,id=dimm1,size=2048M -numa node,memdev=dimm1 -kernel /opt/kata/share/kata-containers/vmlinuz-6.7-136-confidential -append tsc=reliable no_timer_check rcupdate.rcu_expedited=1 i8042.direct=1 i8042.dumbkbd=1 i8042.nopnp=1 i8042.noaux=1 noreplace-smp reboot=k cryptomgr.notests net.ifnames=0 pci=lastbus=0 root=/dev/pmem0p1 rootflags=dax,data=ordered,errors=remount-ro ro rootfstype=ext4 console=hvc0 console=hvc1 quiet systemd.show_status=false panic=1 nr_cpus=4 selinux=0 systemd.unit=kata-containers.target systemd.mask=systemd-networkd.service systemd.mask=systemd-networkd.socket scsi_mod.scan=none -pidfile /run/vc/vm/baabb31ff0c798a31bca7373f2abdbf2936375a5729a3599799c0a225f3b9612/pid -smp 1,cores=1,threads=1,sockets=4,maxcpus=4
```

The launched kernel (`/opt/kata/share/kata-containers/vmlinuz-6.7-136-confidential`) and guest image (`/opt/kata/share/kata-containers/kata-ubuntu-latest-confidential.image`), as well as QEMU (`/opt/kata/bin/qemu-system-x86_64`) were all installed on the host system by the CoCo operator runtime.

If you run `uname -a` inside the coco-demo-01 and compare with the value obtained from the host then you will notice the container is isolated by a different kernel, as shown below:

```shell
$ kubectl exec coco-demo-01 -- uname -a
Linux 6.7.0 #1 SMP Mon Sep  9 09:48:13 UTC 2024 x86_64 GNU/Linux
$ uname -a
Linux coco-demo 5.15.0-97-generic #107-Ubuntu SMP Wed Feb 7 13:26:48 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

If you were running on a platform with supported TEE then you would be able to check if the VM is enabled with confidential features, for example, memory and registers state encryption as well as hardware-based measurement and attestation.

Inside the VM, there is an agent (Kata Agent) process which responds to requests from the Kata Containers runtime to manage the containers' lifecycle. In the next sections, we explain how that agent cooperates with other elements of the architecture to increase the confidentiality of the workload.

### The host cannot see the container image

Oversimplifying, in a normal Kata Containers pod the container image is pulled by the container runtime on the host and is mounted inside the VM. The CoCo implementation changes that behavior through a chain of delegations so that the image is directly pulled from the guest VM, resulting in the host having no access to its content (except for some metadata).

If you have the `ctr` command in your environment then you can check that only the **quay.io/prometheus/busybox**'s manifest was cached in containerd's storage as well as no rootfs directory exists in `/run/kata-containers/shared/sandboxes/<pod id>` as shown below:

```shell
$ sudo ctr -n "k8s.io" image check name==quay.io/prometheus/busybox:latest
REF                           	TYPE                                                  	DIGEST                                                              	STATUS       	SIZE        	UNPACKED
quay.io/prometheus/busybox:latest application/vnd.docker.distribution.manifest.list.v2+json sha256:dfa54ef35e438b9e71ac5549159074576b6382f95ce1a434088e05fd6b730bc4 incomplete (1/3) 1.0 KiB/1.2 MiB false
$ sudo find /run/kata-containers/shared/sandboxes/baabb31ff0c798a31bca7373f2abdbf2936375a5729a3599799c0a225f3b9612/
/run/kata-containers/shared/sandboxes/baabb31ff0c798a31bca7373f2abdbf2936375a5729a3599799c0a225f3b9612/
/run/kata-containers/shared/sandboxes/baabb31ff0c798a31bca7373f2abdbf2936375a5729a3599799c0a225f3b9612/mounts
/run/kata-containers/shared/sandboxes/baabb31ff0c798a31bca7373f2abdbf2936375a5729a3599799c0a225f3b9612/shared
```

It is worth mentioning that not caching on the host has its downside, as images cannot be shared across pods, thus impacting containers brings up performance. This is an area that the CoCo community will be addressing with a better solution in upcoming releases.

## Going towards confidentiality

In this section we will increase the complexity of the pod and the configuration of CoCo to showcase more features.

### Adding Kata Containers agent policies

Points if you noticed on the *coco-demo-01* pod example that the host owner can execute arbitrary commands in the container, potentially stealing sensitive data, which obviously goes against the confidentiality mantra of “never trust the host”. Hopefully this operation and others default behaviors can be configured by using the [Kata Containers agent policy](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-the-kata-agent-policy.md) mechanism.

As an example, let’s show how to block the ExecProcessRequest endpoint of the kata-agent to deny the execution of commands in the container. First you need to encode in base64 a [Rego policy file](https://www.openpolicyagent.org/docs/latest/policy-language/) as shown below:

```shell
$ curl -s https://raw.githubusercontent.com/kata-containers/kata-containers/refs/heads/main/src/kata-opa/allow-all-except-exec-process.rego | base64 -w 0
IyBDb3B5cmlnaHQgKGMpIDIwMjMgTWljcm9zb2Z0IENvcnBvcmF0aW9uCiMKIyBTUERYLUxpY2Vuc2UtSWRlbnRpZmllcjogQXBhY2hlLTIuMAojCgpwYWNrYWdlIGFnZW50X3BvbGljeQoKZGVmYXVsdCBBZGRBUlBOZWlnaGJvcnNSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBBZGRTd2FwUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgQ2xvc2VTdGRpblJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IENvcHlGaWxlUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgQ3JlYXRlQ29udGFpbmVyUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgQ3JlYXRlU2FuZGJveFJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IERlc3Ryb3lTYW5kYm94UmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgR2V0TWV0cmljc1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IEdldE9PTUV2ZW50UmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgR3Vlc3REZXRhaWxzUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgTGlzdEludGVyZmFjZXNSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBMaXN0Um91dGVzUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgTWVtSG90cGx1Z0J5UHJvYmVSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBPbmxpbmVDUFVNZW1SZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBQYXVzZUNvbnRhaW5lclJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFB1bGxJbWFnZVJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFJlYWRTdHJlYW1SZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBSZW1vdmVDb250YWluZXJSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBSZW1vdmVTdGFsZVZpcnRpb2ZzU2hhcmVNb3VudHNSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBSZXNlZWRSYW5kb21EZXZSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBSZXN1bWVDb250YWluZXJSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBTZXRHdWVzdERhdGVUaW1lUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgU2V0UG9saWN5UmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgU2lnbmFsUHJvY2Vzc1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFN0YXJ0Q29udGFpbmVyUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgU3RhcnRUcmFjaW5nUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgU3RhdHNDb250YWluZXJSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBTdG9wVHJhY2luZ1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFR0eVdpblJlc2l6ZVJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFVwZGF0ZUNvbnRhaW5lclJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFVwZGF0ZUVwaGVtZXJhbE1vdW50c1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFVwZGF0ZUludGVyZmFjZVJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFVwZGF0ZVJvdXRlc1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFdhaXRQcm9jZXNzUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgV3JpdGVTdHJlYW1SZXF1ZXN0IDo9IHRydWUKCmRlZmF1bHQgRXhlY1Byb2Nlc3NSZXF1ZXN0IDo9IGZhbHNlCg==
```

Then you pass the policy to the runtime via `io.katacontainers.config.agent.policy` pod annotation. You should create the **coco-demo-02.yaml** file with the content:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: coco-demo-02
  annotations:
	"io.containerd.cri.runtime-handler": "kata-qemu-coco-dev"
	io.katacontainers.config.agent.policy: IyBDb3B5cmlnaHQgKGMpIDIwMjMgTWljcm9zb2Z0IENvcnBvcmF0aW9uCiMKIyBTUERYLUxpY2Vuc2UtSWRlbnRpZmllcjogQXBhY2hlLTIuMAojCgpwYWNrYWdlIGFnZW50X3BvbGljeQoKZGVmYXVsdCBBZGRBUlBOZWlnaGJvcnNSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBBZGRTd2FwUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgQ2xvc2VTdGRpblJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IENvcHlGaWxlUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgQ3JlYXRlQ29udGFpbmVyUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgQ3JlYXRlU2FuZGJveFJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IERlc3Ryb3lTYW5kYm94UmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgR2V0TWV0cmljc1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IEdldE9PTUV2ZW50UmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgR3Vlc3REZXRhaWxzUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgTGlzdEludGVyZmFjZXNSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBMaXN0Um91dGVzUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgTWVtSG90cGx1Z0J5UHJvYmVSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBPbmxpbmVDUFVNZW1SZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBQYXVzZUNvbnRhaW5lclJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFB1bGxJbWFnZVJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFJlYWRTdHJlYW1SZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBSZW1vdmVDb250YWluZXJSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBSZW1vdmVTdGFsZVZpcnRpb2ZzU2hhcmVNb3VudHNSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBSZXNlZWRSYW5kb21EZXZSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBSZXN1bWVDb250YWluZXJSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBTZXRHdWVzdERhdGVUaW1lUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgU2V0UG9saWN5UmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgU2lnbmFsUHJvY2Vzc1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFN0YXJ0Q29udGFpbmVyUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgU3RhcnRUcmFjaW5nUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgU3RhdHNDb250YWluZXJSZXF1ZXN0IDo9IHRydWUKZGVmYXVsdCBTdG9wVHJhY2luZ1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFR0eVdpblJlc2l6ZVJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFVwZGF0ZUNvbnRhaW5lclJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFVwZGF0ZUVwaGVtZXJhbE1vdW50c1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFVwZGF0ZUludGVyZmFjZVJlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFVwZGF0ZVJvdXRlc1JlcXVlc3QgOj0gdHJ1ZQpkZWZhdWx0IFdhaXRQcm9jZXNzUmVxdWVzdCA6PSB0cnVlCmRlZmF1bHQgV3JpdGVTdHJlYW1SZXF1ZXN0IDo9IHRydWUKCmRlZmF1bHQgRXhlY1Byb2Nlc3NSZXF1ZXN0IDo9IGZhbHNlCg==
spec:
  runtimeClassName: kata-qemu-coco-dev
  containers:
	- name: busybox
    image: quay.io/prometheus/busybox:latest
    imagePullPolicy: Always
    command:
      - sleep
      - "infinity"
  restartPolicy: Never
```

Create the pod, wait for it to be RUNNING, then check that kubectl cannot exec in the container. As a matter of comparison, run exec on coco-demo-01 as shown below:

```shell
$ kubectl apply -f coco-demo-02.yaml
pod/coco-demo-02 created
$ kubectl get pod
NAME       	READY   STATUS	RESTARTS   AGE
coco-demo-01   1/1 	Running   0      	16h
coco-demo-02   1/1 	Running   0      	41s
$ kubectl exec coco-demo-02 -- uname -a
error: Internal error occurred: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "69dc5c8c5b36f0afa330e9ddcc022117203493826cfd95c7900807d6cad985fd": cannot enter container 2da76aa4f403fb234507a23d852528d839eca20ad364219306815ca90478e00f, with err rpc error: code = PermissionDenied desc = "ExecProcessRequest is blocked by policy: ": unknown
$ kubectl exec coco-demo-01 -- uname -a
Linux 6.7.0 #1 SMP Mon Sep  9 09:48:13 UTC 2024 x86_64 GNU/Linux
```

Great! Now you know how to set a policy for the kata-agent. You can read more about that feature [here](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-the-kata-agent-policy.md).

But you might be asking yourself: “what if a malicious agent modifies or simply drops the policy annotation?” The short answer is: “let’s attest its integrity!”. In the next section we will be talking about the role of remote attestation on CoCo.

### Attesting them all!

Since the early versions of the CoCo project, attestation has certainly changed and that has received a lot of attention from the community. The attestation area is quite complex to explain it fully in this blog, so we recommend that you read the following documents before continue:

* [Understanding the Confidential Containers Attestation Flow](https://www.redhat.com/en/blog/understanding-confidential-containers-attestation-flow) - slightly outdated but it still shows the big picture in easy language

### Preparing an KBS

In this blog we will be deploying a development/test version of the Key Broker Server (KBS) on the same Kubernetes as CoCo runtime is running. In production, KBS should be running on a trusted environment instead, and possibly you want to use our [trustee operator](https://github.com/confidential-containers/trustee-operator) or maybe even your own implementation.

The following instructions will end up with KBS installed on your cluster and having its service exposed via [nodeport](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport). For further information about deploying the KBS on Kubernetes, see this [README](https://github.com/confidential-containers/trustee/tree/v0.10.1/kbs/config/kubernetes). So do:

```shell
$ git clone https://github.com/confidential-containers/trustee --single-branch -b v0.10.1
$ cd trustee/kbs/config/kubernetes
$ echo "somesecret" > overlays/$(uname -m)/key.bin
$ export DEPLOYMENT_DIR=nodeport
$ ./deploy-kbs.sh
<output omitted>
$ export KBS_PRIVATE_KEY="${PWD}/base/kbs.key"
```

Wait the KBS deployment be ready and running just like below:

```shell
$ kubectl -n coco-tenant get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
kbs	1/1 	1        	1       	26m
```

You will need the KBS host and port to configure the pod. These values can be obtained like in below listing:

```shell
$ export KBS_HOST=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}' -n coco-tenant)
$ export KBS_PORT=$(kubectl get svc "kbs" -n "coco-tenant" -o jsonpath='{.spec.ports[0].nodePort}')
```

At this point KBS is up and running but lacks policies and resources. To facilitate its configuration we will be using the [kbs-client](https://github.com/confidential-containers/trustee/tree/v0.10.1/tools/kbs-client) tool.  Use the [ORAS tool](https://oras.land) to download a build of kbs-client:

```shell
$ curl -LOs "https://github.com/oras-project/oras/releases/download/v1.2.0/oras_1.2.0_linux_amd64.tar.gz"
$ tar xvzf oras_1.2.0_linux_amd64.tar.gz
$ ./oras pull ghcr.io/confidential-containers/staged-images/kbs-client:sample_only-x86_64-linux-gnu-68607d4300dda5a8ae948e2562fd06d09cbd7eca
$ chmod +x kbs-client
```

The KBS address is passed to the kata-agent within the confidential VM via `kernel_params` annotation (`io.katacontainers.config.hypervisor.kernel_params`). You should set the `agent.aa_kbc_params` parameter to `cc_kbc::http://host:port` where host:port represents the address.

As an example, create **coco-demo-03.yaml** with the content below but replace `http://192.168.122.153:31491` with the `http://$KBS_HOST:$KBS_PORT`  value that matches the KBS address of your installation:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: coco-demo-03
  annotations:
	"io.containerd.cri.runtime-handler": "kata-qemu-coco-dev"
	io.katacontainers.config.hypervisor.kernel_params: " agent.aa_kbc_params=cc_kbc::http://192.168.122.153:31491"
spec:
  runtimeClassName: kata-qemu-coco-dev
  containers:
	- name: busybox
    image: quay.io/prometheus/busybox:latest
    imagePullPolicy: Always
    command:
      - sleep
      - "infinity"
  restartPolicy: Never
```

If you apply **coco-demo-03.yaml** then the pod should run as expected, but nothing interesting really (apparently) happens. However, we have everything in place to dive in some features on the next sections.

### Getting resources from the Confidential Data Hub

The [Confidential Data Hub](https://github.com/confidential-containers/guest-components/tree/v0.10.0/confidential-data-hub) (CDH) is a service running inside the confidential VM that shares the same networking namespace of the pod, meaning that its REST API can be accessed from within a confidential container. Among the services provided by CDH, there is the [sealed secrets unsealing](https://github.com/confidential-containers/guest-components/blob/v0.10.0/confidential-data-hub/docs/SEALED_SECRET.md) (not covered on this blog) and [getting resources from KBS](https://github.com/confidential-containers/guest-components/blob/v0.10.0/confidential-data-hub/docs/RESOURCES_SERVICES.md). In this section we will show the latter as a tool for learning more about the attestation flow.

Let’s add the to-be-fetched resource to the KBS first. Think of that resource as a secret key required to unencrypt an important file for data processing. Using `kbs-client`, do the following (`KBS_HOST`, `KBS_PORT` and `KBS_PRIVATE_KEY` are previously defined variables):

```shell
$ echo "MySecretKey" > secret.txt
$ ./kbs-client --url "http://$KBS_HOST:$KBS_PORT" config --auth-private-key "$KBS_PRIVATE_KEY" set-resource --path default/secret/1 --resource-file secret.txt
Set resource success
 resource: TXlTZWNyZXRLZXkK
```

The CDH service listens at `127.0.0.1:8006` address. So you just need to modify the **coco-demo-03.yaml** file to run the `wget -O- http://127.0.0.1:8006/cdh/resource/default/secret/1` command:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: coco-demo-04
  annotations:
	"io.containerd.cri.runtime-handler": "kata-qemu-coco-dev"
	io.katacontainers.config.hypervisor.kernel_params: " agent.aa_kbc_params=cc_kbc::http://192.168.122.153:31491"
spec:
  runtimeClassName: kata-qemu-coco-dev
  containers:
	- name: busybox
    image: quay.io/prometheus/busybox:latest
    imagePullPolicy: Always
    command:
      - sh
      - -c
      - |
        wget -O- http://127.0.0.1:8006/cdh/resource/default/secret/1; sleep infinity
  restartPolicy: Never
```

Apply **coco-demo-04.yaml** and wait for it to get into `RUNNING` state. Checking the pod logs you will notice that wget failed to fetch the secret:

```shell
$ kubectl apply -f coco-demo-04.yaml
pod/coco-demo-04 created
$ kubectl wait --for=condition=Ready pod/coco-demo-04
pod/coco-demo-04 condition met
$ kubectl logs pod/coco-demo-04
Connecting to 127.0.0.1:8006 (127.0.0.1:8006)
wget: server returned error: HTTP/1.1 500 Internal Server Error
```

Looking at the KBS logs we can find that the problem was caused by `Resource not permitted` denial:

```shell
$ kubectl logs -l app=kbs -n coco-tenant
Defaulted container "kbs" out of: kbs, copy-config (init)
[2024-11-07T20:04:32Z INFO  kbs::http::resource] Get resource from kbs:///default/secret/1
[2024-11-07T20:04:32Z ERROR kbs::http::error] Resource not permitted.
[2024-11-07T20:04:32Z INFO  actix_web::middleware::logger] 10.244.0.1 "GET /kbs/v0/resource/default/secret/1 HTTP/1.1" 401 112 "-" "attestation-agent-kbs-client/0.1.0" 0.000351
[2024-11-07T20:04:32Z INFO  kbs::http::attest] Auth API called.
[2024-11-07T20:04:32Z INFO  actix_web::middleware::logger] 10.244.0.1 "POST /kbs/v0/auth HTTP/1.1" 200 74 "-" "attestation-agent-kbs-client/0.1.0" 0.000125
[2024-11-07T20:04:32Z INFO  kbs::http::attest] Attest API called.
[2024-11-07T20:04:32Z INFO  attestation_service] Sample Verifier/endorsement check passed.
[2024-11-07T20:04:32Z INFO  attestation_service] Policy check passed.
[2024-11-07T20:04:32Z INFO  attestation_service] Attestation Token (Simple) generated.
[2024-11-07T20:04:32Z INFO  actix_web::middleware::logger] 10.244.0.1 "POST /kbs/v0/attest HTTP/1.1" 200 2171 "-" "attestation-agent-kbs-client/0.1.0" 0.001940
```

You need to configure a permissive resources policy in the KBS because you aren’t running on a real TEE, hence the attestation verification failed. See some [sample policies](https://github.com/confidential-containers/trustee/tree/v0.10.1/kbs/sample_policies) for more examples. Create an sample permissive **resources_policy.rego** file (Note: do not use this in production as it doesn’t validate confidential hardware) with content:

```rego
package policy

default allow = false

allow {
    input["tee"] == "sample"
}
```

The `GetResource` request to CDH is an attested operation. The policy in **resources_policy.rego** will deny access to any resources by default, but release it in case the request came from a sample TEE.

Apply the resources_policy.rego policy to the KBS, then respin the coco-demo-04 pod, and you will see `MySecretKey` is now fetched:

```shell
$ ./kbs-client --url "http://$KBS_HOST:$KBS_PORT" config --auth-private-key "$KBS_PRIVATE_KEY" set-resource-policy --policy-file resources_policy.rego
Set resource policy success
 policy: cGFja2FnZSBwb2xpY3kKCmRlZmF1bHQgYWxsb3cgPSBmYWxzZQoKYWxsb3cgewogICAgaW5wdXRbInRlZSJdID09ICJzYW1wbGUiCn0K
$ kubectl apply -f coco-demo-04.yaml
pod/coco-demo-04 created
$ kubectl wait --for=condition=Ready pod/coco-demo-04
pod/coco-demo-04 condition met
$ kubectl logs pod/coco-demo-04
Connecting to 127.0.0.1:8006 (127.0.0.1:8006)
writing to stdout
-                	100% |********************************|	12  0:00:00 ETA
written to stdout
MySecretKey
```

In the KBS log messages below you can see that the Attestation Service (AS) was involved in the request. A sample verifier was invoked in the place of a real hardware-oriented one for the sake of emulating the verification process. The generated attestation token (see `Attestation Token (Simple) generated` message in the log) is passed all the way back to the CDH on the confidential VM, which then can finally request the resource (the `Get resource from kbs:///default/secret/1` message) from KBS.

```shell
$ kubectl logs -l app=kbs -n coco-tenant
Defaulted container "kbs" out of: kbs, copy-config (init)
[2024-11-07T22:04:22Z INFO  actix_web::middleware::logger] 10.244.0.1 "POST /kbs/v0/auth HTTP/1.1" 200 74 "-" "attestation-agent-kbs-client/0.1.0" 0.000185
[2024-11-07T22:04:22Z INFO  kbs::http::attest] Attest API called.
[2024-11-07T22:04:22Z INFO  attestation_service] Sample Verifier/endorsement check passed.
[2024-11-07T22:04:22Z INFO  attestation_service] Policy check passed.
[2024-11-07T22:04:22Z INFO  attestation_service] Attestation Token (Simple) generated.
[2024-11-07T22:04:22Z INFO  actix_web::middleware::logger] 10.244.0.1 "POST /kbs/v0/attest HTTP/1.1" 200 2171 "-" "attestation-agent-kbs-client/0.1.0" 0.001931
[2024-11-07T22:04:22Z WARN  kbs::token::coco] No Trusted Certificate in Config, skip verification of JWK cert of Attestation Token
[2024-11-07T22:04:22Z INFO  kbs::http::resource] Get resource from kbs:///default/secret/1
[2024-11-07T22:04:22Z INFO  kbs::http::resource] Resource access request passes policy check.
[2024-11-07T22:04:22Z INFO  actix_web::middleware::logger] 10.244.0.1 "GET /kbs/v0/resource/default/secret/1 HTTP/1.1" 200 504 "-" "attestation-agent-kbs-client/0.1.0" 0.001104
```

What we have shown in this section is obviously a simplification of the workflows involving the guest (CDH, attestation agent, etc..) and server-side (KBS, attestation service, etc…) components.  The most important lesson here is that even if you are running from your workstation without any TEE hardware, the workflows will still be exercised just by using a sample attestation verifier that always returns true.

### Using encrypted container images

Let us show one more example of another feature that you can give it a try on your workstation; support for running workloads from encrypted container images.

For this example we will use an image already encrypted (**ghcr.io/confidential-containers/test-container:multi-arch-encrypted**) that is used in CoCo’s development tests. Create the **coco-demo-05.yaml** file with following content:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: coco-demo-05
  annotations:
	"io.containerd.cri.runtime-handler": "kata-qemu-coco-dev"
	io.katacontainers.config.hypervisor.kernel_params: " agent.aa_kbc_params=cc_kbc::http://192.168.122.153:31491"
spec:
  runtimeClassName: kata-qemu-coco-dev
  containers:
    - name: ssh-demo
      image: ghcr.io/confidential-containers/test-container:multi-arch-encrypted
      imagePullPolicy: Always
      command:
        - sleep
        - "infinity"
  restartPolicy: Never
```

Apply the pod, wait a little bit and you will see it failed to start with `StartError` status:

```shell
$ kubectl describe pods/coco-demo-05
Name:            	coco-demo-05
Namespace:       	default
Priority:        	0
Runtime Class Name:  kata-qemu-coco-dev
Service Account: 	default
Node:            	coco-demo/192.168.122.153
Start Time:      	Mon, 11 Nov 2024 15:39:08 +0000
Labels:          	<none>
Annotations:     	io.containerd.cri.runtime-handler: kata-qemu-coco-dev
                 	io.katacontainers.config.hypervisor.kernel_params:  agent.aa_kbc_params=cc_kbc::http://192.168.122.153:31491
Status:          	Failed
IP:              	10.244.0.15
IPs:
  IP:  10.244.0.15
Containers:
  ssh-demo:
	Container ID:  containerd://a94522e7f9ed08b9384874be7b696fbd25998ed0fc24f7c13fc7f8167fb06c80
	Image:     	ghcr.io/confidential-containers/test-container:multi-arch-encrypted
	Image ID:  	ghcr.io/confidential-containers/test-container@sha256:96d19d2729d83379c8ddc6b2b9551d2dbe6797632c6eb7e6f50cbadc283bfdf6
	Port:      	<none>
	Host Port: 	<none>
	Command:
  	sleep
  	infinity
	State:  	Terminated
  	Reason:   StartError
  	Message:  failed to create containerd task: failed to create shim task: failed to handle layer: failed to get decrypt key

Caused by:
	no suitable key found for decrypting layer key:
 	keyprovider: failed to unwrap key by ttrpc

Stack backtrace:
   0: <unknown>
   1: <unknown>
   2: <unknown>
   3: <unknown>
   4: <unknown>

Stack backtrace:
   0: <unknown>
   1: <unknown>
   2: <unknown>
   3: <unknown>
   4: <unknown>
   5: <unknown>
   6: <unknown>
   7: <unknown>
   8: <unknown>
   9: <unknown>
  10: <unknown>
  11: <unknown>
  12: <unknown>
  13: <unknown>
  14: <unknown>
  15: <unknown>: unknown
  	Exit Code:	128
  	Started:  	Thu, 01 Jan 1970 00:00:00 +0000
  	Finished: 	Mon, 11 Nov 2024 15:39:19 +0000
	Ready:      	False
	Restart Count:  0
	Environment:	<none>
	Mounts:
  	/var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-qkvv4 (ro)
Conditions:
  Type                    	Status
  PodReadyToStartContainers   False
  Initialized             	True
  Ready                   	False
  ContainersReady         	False
  PodScheduled            	True
Volumes:
  kube-api-access-qkvv4:
	Type:                	Projected (a volume that contains injected data from multiple sources)
	TokenExpirationSeconds:  3607
	ConfigMapName:       	kube-root-ca.crt
	ConfigMapOptional:   	<nil>
	DownwardAPI:         	true
QoS Class:               	BestEffort
Node-Selectors:          	katacontainers.io/kata-runtime=true
Tolerations:             	node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                         	node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type 	Reason 	Age	From           	Message
  ---- 	------ 	----   ----           	-------
  Normal   Scheduled  2m42s  default-scheduler  Successfully assigned default/coco-demo-05 to coco-demo
  Normal   Pulling	2m39s  kubelet        	Pulling image "ghcr.io/confidential-containers/test-container:multi-arch-encrypted"
  Normal   Pulled 	2m35s  kubelet        	Successfully pulled image "ghcr.io/confidential-containers/test-container:multi-arch-encrypted" in 3.356s (3.356s including waiting). Image size: 5581550 bytes.
  Normal   Created	2m35s  kubelet        	Created container ssh-demo
  Warning  Failed 	2m32s  kubelet        	Error: failed to create containerd task: failed to create shim task: failed to handle layer: failed to get decrypt key

Caused by:
	no suitable key found for decrypting layer key:
 	keyprovider: failed to unwrap key by ttrpc

Stack backtrace:
   0: <unknown>
   1: <unknown>
   2: <unknown>
   3: <unknown>
   4: <unknown>

Stack backtrace:
   0: <unknown>
   1: <unknown>
   2: <unknown>
   3: <unknown>
   4: <unknown>
   5: <unknown>
   6: <unknown>
   7: <unknown>
   8: <unknown>
   9: <unknown>
  10: <unknown>
  11: <unknown>
  12: <unknown>
  13: <unknown>
  14: <unknown>
  15: <unknown>: unknown
```

The reason why it failed is because the decryption key wasn’t found in the KBS. So let’s insert the key:

```shell
$ echo "HUlOu8NWz8si11OZUzUJMnjiq/iZyHBJZMSD3BaqgMc=" | base64 -d > image_key.txt
$ ./kbs-client --url "http://$KBS_HOST:$KBS_PORT" config --auth-private-key "$KBS_PRIVATE_KEY" set-resource --path default/key/ssh-demo --resource-file image_key.txt
Set resource success
 resource: HUlOu8NWz8si11OZUzUJMnjiq/iZyHBJZMSD3BaqgMc=
```

Then restart the coco-demo-05 pod and it should get running just fine.

As demonstrated by the listing below, you can inspect the image with skopeo. Note that each of its layers is encrypted (`MIMEType` is `tar+gzip+encrypted`) and annotated with `org.opencontainers.image.enc.*` tags. In particular, the `org.opencontainers.image.enc.keys.provider.attestation-agent` annotation encodes the decryption key path (e.g. `kbs:///default/key/ssh-demo`) in the KBS:

```shell
$ skopeo inspect --raw docker://ghcr.io/confidential-containers/test-container:multi-arch-encrypted
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.oci.image.index.v1+json",
   "manifests": [
    {
     	"mediaType": "application/vnd.oci.image.manifest.v1+json",
     	"size": 4976,
     	"digest": "sha256:ac0ed3364d54120a1025c74f555b21cb378d4d0f62a398c5ce3e1b89fa4ca637",
     	"platform": {
          "architecture": "amd64",
          "os": "linux"
     	}
  	},
    {
     	"mediaType": "application/vnd.oci.image.manifest.v1+json",
     	"size": 4976,
     	"digest": "sha256:b9b29ea19f50c2ca814d4e0b72d1183c474b1e6d75cee61fb1ac6777bc38f688",
     	"platform": {
          "architecture": "s390x",
          "os": "linux"
    }
    }
   ]
$ skopeo inspect --raw docker://ghcr.io/confidential-containers/test-container@sha256:ac0ed3364d54120a1025c74f555b21cb378d4d0f62a398c5ce3e1b89fa4ca637 | jq -r '.layers[0]'
{
  "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip+encrypted",
  "digest": "sha256:766192d6b7d02409620f7c1f09bfa11fc273a2ec2e6383a56fb5fcf52d94d73e",
  "size": 2829647,
  "annotations": {
	"org.opencontainers.image.enc.keys.provider.attestation-agent": "eyJraWQiOiJrYnM6Ly8vZGVmYXVsdC9rZXkvc3NoLWRlbW8iLCJ3cmFwcGVkX2RhdGEiOiJxdkY5Rkh4eXZFRDZBd21IYXpGOTV1d01MUjdNb2ZoUHRHaGE4MWZyRGNrb21UNEUvekFkSHB4c1ZJcnpud0JKR3J1bVB3NDFIVUpwN0RNME9qUzlmMFhtQ1dMUWtTYkZ1Y280eGQyMFdoMFBreDdJQmdGTDduMnhKbkZiQ2V5NFNhRktXZDZ4MDJRNVd5VkVvekU3V1h1R2wwaHVNVGJHb3UxV3JIa0FzTVZwRTlYejVGcVNoTlMvUFZ4aTAyUTVGK2d6RGJZRENIb2crZ2ZqQlhNTkdlS2hNSXF6ZnM0LzhSUDRzZ1RaelV3Z3ZXMTFKUmQ2WVhHM1ZySW01NGVxbW1Pci94OG8yM2hFakIvWS85TzhvTHc9IiwiaXYiOiJoTXNWcE1ZZXRwWUtOK0pzIiwid3JhcF90eXBlIjoiQTI1NkdDTSJ9",
	"org.opencontainers.image.enc.pubopts": "eyJjaXBoZXIiOiJBRVNfMjU2X0NUUl9ITUFDX1NIQTI1NiIsImhtYWMiOiJCWXg2dUg2MEZMc2lNMUc2RUk4KzZzTlQ5QlRMN2lVamtvRWlmNVVBM09nPSIsImNpcGhlcm9wdGlvbnMiOnt9fQ=="
  }
}
$ skopeo inspect --raw docker://ghcr.io/confidential-containers/test-container@sha256:ac0ed3364d54120a1025c74f555b21cb378d4d0f62a398c5ce3e1b89fa4ca637 | jq -r '.layers[0].annotations["org.opencontainers.image.enc.keys.provider.attestation-agent"]' | base64 -d
{"kid":"kbs:///default/key/ssh-demo","wrapped_data":"qvF9FHxyvED6AwmHazF95uwMLR7MofhPtGha81frDckomT4E/zAdHpxsVIrznwBJGrumPw41HUJp7DM0OjS9f0XmCWLQkSbFuco4xd20Wh0Pkx7IBgFL7n2xJnFbCey4SaFKWd6x02Q5WyVEozE7WXuGl0huMTbGou1WrHkAsMVpE9Xz5FqShNS/PVxi02Q5F+gzDbYDCHog+gfjBXMNGeKhMIqzfs4/8RP4sgTZzUwgvW11JRd6YXG3VrIm54eqmmOr/x8o23hEjB/Y/9O8oLw=","iv":"hMsVpMYetpYKN+Js","wrap_type":"A256GCM"}
```

If you are curious about encryption of container images for CoCo, please refer to the [Keyprovider tool](https://github.com/confidential-containers/guest-components/tree/v0.10.0/attestation-agent/coco_keyprovider).

## Closing remarks

There are many features not covered in this blog that in common leverage the attestation mechanism implemented in CoCo. All these features can be tested/fixed/developed in a workstation without confidential hardware as long as the `kata-qemu-coco-dev` runtimeclass is employed. Thanks to all the sample and mocking implementations that our community built to overcome the dependency on specialized hardware.

## Summary

In this tutorial, we have taken you through the process of deploying CoCo on a Kubernetes cluster and creating your first pod.

We have installed CoCo with a special runtime that allows you to create the pod with an encrypted image support but without having to use any confidential hardware. We also showed you some fundamental concepts of attestation and high level details of its implementation.
