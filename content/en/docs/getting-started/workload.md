---
title: Simple Workload
description: Running a simple confidential workload 
weight: 30
categories:
- getting-started
---

## Creating a sample Confidential Containers workload

Once you've used the operator to install Confidential Containers, you can run a pod with CoCo by simply adding a runtime class.
First, we will use the `kata-qemu-coco-dev` runtime class which uses CoCo without hardware support.
Initially we will try this with an unencrypted container image.

In this example, we will be using the bitnami/nginx image as described in the following yaml:
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
  annotations:
    io.containerd.cri.runtime-handler: kata-qemu-coco-dev
spec:
  containers:
  - image: bitnami/nginx:1.22.0
    name: nginx
  dnsPolicy: ClusterFirst
  runtimeClassName: kata-qemu-coco-dev
```

For the most basic workloads, setting the `runtimeClassName` and `runtime-handler` annotation is usually
the only requirement for the pod YAML.

Create a pod YAML file as previously described (we named it `nginx.yaml`) .

Create the workload:
```
kubectl apply -f nginx.yaml
```
Output:
```
pod/nginx created
```

Ensure the pod was created successfully (in running state):
```
kubectl get pods
```
Output:
```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          3m50s
```
