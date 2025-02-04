---
title: Container Launch with SNP Memory Encryption
description: How to launch a container with SNP Memory Encryption
categories:
- examples
tags:
- container launch
- snp
---

# Container Launch With Memory Encryption

## Launch a Confidential Service

To launch a container with SNP memory encryption, the SNP runtime class (`kata-qemu-snp`) must be specified. A base alpine docker container ([Dockerfile](https://github.com/kata-containers/kata-containers/blob/main/tests/integration/kubernetes/runtimeclass_workloads/confidential/unencrypted/Dockerfile)) has been previously built for testing purposes. This image has also been prepared with SSH access and provisioned with a [SSH public key](https://github.com/kata-containers/kata-containers/blob/main/tests/integration/kubernetes/runtimeclass_workloads/confidential/unencrypted/ssh/unencrypted.pub) for validation purposes.

Here is a sample service yaml specifying the SNP runtime class: 

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

Save the contents of this yaml to a file called `confidential-unencrypted.yaml`.

Start the service:
 
```shell
kubectl apply -f confidential-unencrypted.yaml
```

Check for errors:

```shell
kubectl describe pod confidential-unencrypted
```

If there are no errors in the Events section, then the container has been successfully created with SNP memory encryption.

## Validate SNP Memory Encryption

The container `dmesg` log can be parsed to indicate that SNP memory encryption is enabled and active. The container image defined in the yaml sample above was built with a predefined key that is authorized for SSH access.

Get the pod IP:

```shell
pod_ip=$(kubectl get pod -o wide | grep confidential-unencrypted | awk '{print $6;}')
```

Download and save the SSH private key and set the permissions.

```shell
wget https://github.com/kata-containers/kata-containers/raw/main/tests/integration/kubernetes/runtimeclass_workloads/confidential/unencrypted/ssh/unencrypted -O confidential-image-ssh-key

chmod 600 confidential-image-ssh-key
```

The following command will run a remote SSH command on the container to check if SNP memory encryption is active:

```shell
ssh -i confidential-image-ssh-key \
  -o "StrictHostKeyChecking no" \
  -t root@${pod_ip} \
  'dmesg | grep "Memory Encryption Features""'
```

If SNP is enabled and active, the output should return:

```shell
[    0.150045] Memory Encryption Features active: AMD SNP
```
