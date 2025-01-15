---
title: Cluster Setup
description: Cluster prerequisites 
weight: 20
categories:
- prerequisites
tags:
- k8s
---

Confidential Containers requires Kubernetes.
A cluster must be installed before running the operator.
Many different clusters can be used but they should meet the following requirements.
- The minimum Kubernetes version is 1.24
- Cluster must use `containerd` or `cri-o`.
- At least one node has the label `node.kubernetes.io/worker`.
- SELinux is not enabled.

If you use Minikube or Kind to setup your cluster, you will only be able to use
runtime classes based on Cloud Hypervisor due to an issue with QEMU.

