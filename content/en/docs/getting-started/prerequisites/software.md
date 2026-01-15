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
A cluster must be installed before installing the Helm charts.
Many different clusters can be used but they should meet the following requirements.
- The minimum Kubernetes version is 1.24
- Cluster must use `containerd`. Note: `cri-o` is not tested with the Helm charts for baremetal deployments.
- At least one node has the label `node.kubernetes.io/worker`.
- SELinux is not enabled.
- Helm 3.8+ is installed.

{{% alert title="Note" color="warning" %}}
Kind and Minikube are not tested anywhere in the project, and those are not encouraged to be used as QEMU is known to **not** work with them.
{{% /alert %}}
