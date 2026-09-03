---
title: 使用 SNP 内存加密启动容器
description: 如何使用 SNP 内存加密启动容器
categories:
- examples
tags:
- container launch
- snp
---

> **说明：** 本文为英文文档的中文译版，英文原版请参见 [Container Launch with SNP Memory Encryption](https://confidentialcontainers.org/docs/examples/snp-container-launch/)。

# 使用 SNP 内存加密启动容器

## 启动机密服务

要以 SNP 内存加密方式启动容器，需要显式指定 SNP runtime class，也就是 `kata-qemu-snp`。这里使用一个提前构建好的 alpine 测试镜像（[Dockerfile](https://github.com/kata-containers/kata-containers/blob/main/tests/integration/kubernetes/runtimeclass_workloads/confidential/unencrypted/Dockerfile)）。这个镜像已经启用 SSH，并预置了一个用于验证的 [SSH 公钥](https://github.com/kata-containers/kata-containers/blob/main/tests/integration/kubernetes/runtimeclass_workloads/confidential/unencrypted/ssh/unencrypted.pub)。

下面是一个指定 SNP runtime class 的示例 Service 和 Deployment：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: "confidential-unencrypted"
spec:
  selector:
    app: "confidential-unencrypted"
  ports:
  - port: 22
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: "confidential-unencrypted"
spec:
  selector:
    matchLabels:
      app: "confidential-unencrypted"
  template:
    metadata:
      labels:
        app: "confidential-unencrypted"
      annotations:
        io.containerd.cri.runtime-handler: kata-qemu-snp
    spec:
      runtimeClassName: kata-qemu-snp
      containers:
      - name: "confidential-unencrypted"
        image: ghcr.io/kata-containers/test-images:unencrypted-nightly
        imagePullPolicy: Always
```

把这段 YAML 保存为 `confidential-unencrypted.yaml`。

启动服务：

```bash
kubectl apply -f confidential-unencrypted.yaml
```

检查是否有错误：

```bash
kubectl describe pod confidential-unencrypted
```

如果 Events 区域没有报错，说明容器已经成功以 SNP 内存加密方式启动。

## 验证 SNP 内存加密

可以通过容器内的 `dmesg` 输出来确认 SNP 内存加密是否已经启用。上面的示例镜像内置了对应私钥所匹配的 SSH 公钥，因此可以直接远程登录验证。

先获取 Pod IP：

```bash
pod_ip=$(kubectl get pod -o wide | grep confidential-unencrypted | awk '{print $6;}')
```

下载 SSH 私钥并设置权限：

```bash
wget https://github.com/kata-containers/kata-containers/raw/main/tests/integration/kubernetes/runtimeclass_workloads/confidential/unencrypted/ssh/unencrypted -O confidential-image-ssh-key

chmod 600 confidential-image-ssh-key
```

下面这条命令会通过 SSH 到容器内部执行命令，检查 SNP 内存加密是否生效：

```bash
ssh -i confidential-image-ssh-key \
  -o "StrictHostKeyChecking no" \
  -t root@${pod_ip} \
  'dmesg | grep "Memory Encryption Features"'
```

如果 SNP 已经启用，输出应类似：

```text
[    0.150045] Memory Encryption Features active: AMD SNP
```
