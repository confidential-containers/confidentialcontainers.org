---
title: CCv0 Operator Demo
date: 2023-01-05
description: >
  The demo shows CCv0 Kata runtime installation and configuration using the coco-operator.
categories:
- demo
tags:
- coco-operator
- demo
---

## Demo Video

[Watch the demo in youtube](https://www.youtube.com/watch?v=4cM3IhfnJLQ)

## Demo Environment setup

### Kubernetes cluster

Setup a two nodes Kubernetes cluster using Ubuntu 20.04. You can use your preferred Kubernetes setup tool. Here is an example using [kcli](https://kcli.readthedocs.io/en/latest/).

Download ubuntu 20.04 image if not present by running the following command:

```bash
kcli download image ubuntu2004
```

Install the cluster:

```bash
kcli create kube generic -P image=ubuntu2004 -P workers=1 testk8s
```

### Replace containerd

Replace containerd on the worker node by building a new containerd from the following branch: [https://github.com/confidential-containers/containerd/tree/CC-main](https://github.com/confidential-containers/containerd/tree/CC-main) ([build instructions](https://github.com/confidential-containers/containerd/blob/CC-main/BUILDING.md))

Modify systemd configuration to use the new binary and restart `containerd` and `kubelet`.

### Verify if the cluster nodes are all up

```bash
kubectl get nodes
```

Sample output from the demo environment:

```console
$ kubectl get nodes
NAME                  STATUS   ROLES                  AGE   VERSION
cck8s-demo-master-0   Ready    control-plane,master   25d   v1.22.3
cck8s-demo-worker-0   Ready    worker                 25d   v1.22.3
```

Make sure at least one Kubernetes node in the cluster has the label `node.kubernetes.io/worker=`.

```bash
kubectl label node $NODENAME node.kubernetes.io/worker=
```

## Operator Setup

```bash
RELEASE_VERSION="main"
kubectl apply -k "github.com/confidential-containers/operator/config/release?ref=${RELEASE_VERSION}"
```

The operator installs everything under the `confidential-containers-system` namespace:

Verify if the operator is running by running the following command:

```bash
kubectl get pods -n confidential-containers-system
```

Sample output from the demo environment:

```console
$ kubectl get pods -n confidential-containers-system
NAME                                              READY   STATUS    RESTARTS   AGE
cc-operator-controller-manager-7f8d6dd988-t9zdm   2/2     Running   0          13s
```

## Confidential Containers Runtime setup

Creating a `CCruntime` object sets up the container runtime. The default payload image sets up the CCv0 demo image of the kata-containers runtime.

```bash
RELEASE_VERSION="main"
kubectl apply -k "github.com/confidential-containers/operator/config/samples/ccruntime/default?ref=${RELEASE_VERSION}"
```

This will create an install daemonset targeting the worker nodes for installation. You can verify the status under the `confidential-containers-system` namespace.

```console
$ kubectl get pods -n confidential-containers-system
NAME                                              READY   STATUS    RESTARTS   AGE
cc-operator-controller-manager-7f8d6dd988-t9zdm   2/2     Running   0          82s
cc-operator-daemon-install-p9ntc                  1/1     Running   0          45s
```

On successful installation, you'll see the following `runtimeClasses` being setup:

```console
$ kubectl get runtimeclasses.node.k8s.io
NAME        HANDLER     AGE
kata        kata        92s
kata-cc     kata-cc     92s
kata-qemu   kata-qemu   92s
```

`kata-cc` runtimeclass uses CCv0 specific configurations.

Now you can deploy the PODs targeting the specific runtimeclasses. The [SSH demo](/docs/demos/ssh-demo) can be used as a compatible workload.
