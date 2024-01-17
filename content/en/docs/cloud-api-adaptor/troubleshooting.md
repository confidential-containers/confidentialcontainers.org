---
title: Cloud API Adaptor Troubleshooting
description: Generic troubleshooting steps after installation of Cloud API Adaptor
weight: 1
categories:
- docs
tags:
- docs
- caa
---

## Application pod created but it stays in `ContainerCreating` state

Let's start by looking at the pods deployed in the `confidential-containers-system` namespace:

```console
$ kubectl get pods -n confidential-containers-system -o wide
NAME                                              READY   STATUS    RESTARTS   AGE   IP            NODE                                NOMINATED NODE   READINESS GATES
cc-operator-controller-manager-76755f9c96-pjj92   2/2     Running   0          1h    10.244.0.14   aks-nodepool1-22620003-vmss000000   <none>           <none>
cc-operator-daemon-install-79c2b                  1/1     Running   0          1h    10.244.0.16   aks-nodepool1-22620003-vmss000000   <none>           <none>
cc-operator-pre-install-daemon-gsggj              1/1     Running   0          1h    10.244.0.15   aks-nodepool1-22620003-vmss000000   <none>           <none>
cloud-api-adaptor-daemonset-2pjbb                 1/1     Running   0          1h    10.224.0.4    aks-nodepool1-22620003-vmss000000   <none>           <none>
```

It is possible that the `cloud-api-adaptor-daemonset` is not deployed correctly. To see what is wrong with it run the following command and look at the events to get insights:

```console
$ kubectl -n confidential-containers-system describe ds cloud-api-adaptor-daemonset
Name:           cloud-api-adaptor-daemonset
Selector:       app=cloud-api-adaptor
Node-Selector:  node-role.kubernetes.io/worker=
...
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  8m13s  daemonset-controller  Created pod: cloud-api-adaptor-daemonset-2pjbb
```

But if the `cloud-api-adaptor-daemonset` is up and in the `Running` state, like shown above then look at the pods' logs, for more insights:

```bash
kubectl -n confidential-containers-system logs daemonset/cloud-api-adaptor-daemonset
```

> **Note**: This is a single node cluster. So there is only one pod named `cloud-api-adaptor-daemonset-*`. But if you are running on a multi-node cluster then look for the node your workload fails to come up and only see the logs of corresponding CAA pod.

If the problem hints that something is wrong with the configuration then look at the configmaps or secrets needed to run CAA:

```bash
$ kubectl -n confidential-containers-system get cm
NAME                         DATA   AGE
cc-operator-manager-config   1      1h
kube-root-ca.crt             1      1h
peer-pods-cm                 7      1h
```

```bash
$ kubectl -n confidential-containers-system get secret
NAME               TYPE     DATA   AGE
peer-pods-secret   Opaque   0      1h
ssh-key-secret     Opaque   1      1h
```
