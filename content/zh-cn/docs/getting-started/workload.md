---
title: 示例工作负载
description: 运行一个示例的机密工作负载
weight: 30
categories:
- getting-started
---

## 创建示例 Confidential Containers 工作负载

使用 Helm Chart 安装 Confidential Containers 后，只需为 Pod 添加一个 RuntimeClass，
即可运行 CoCo 工作负载。
我们使用 `kata-qemu-coco-dev` RuntimeClass，它会在**不依赖机密计算硬件支持**的情况下运行 CoCo。
首先，使用一个未加密的容器镜像进行演示。

本示例使用如下 YAML 所示的 `nginx` 镜像：

```yaml
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
  - name: nginx
    image: nginx:1.29.4
  dnsPolicy: ClusterFirst
  runtimeClassName: kata-qemu-coco-dev
```

对于大多数基础的工作负载来说，通常只需在 Pod YAML 中设置 `runtimeClassName`
和 `runtime-handler` 注解即可。

按照上面的内容创建一个 Pod YAML 文件（本示例命名为 `nginx.yaml`）。

创建工作负载：

```bash
kubectl apply -f nginx.yaml
```

输出示例：

```text
pod/nginx created
```

确认 Pod 已成功创建并处于运行状态：

```bash
kubectl get pods
```

输出示例：

```text
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          3m50s
```