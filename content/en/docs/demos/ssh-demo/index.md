---
date: 2022-11-06
title: SSH Demo
linkTitle: SSH Demo
description: >
  SSH Demo to showcase encrypted memory provided by the TEE
categories:
- demo
tags:
- demo
---

To demonstrate confidential containers capabilities, we run a pod with SSH public key authentication.

Compared to the execution of and login to a shell on a pod, an SSH connection is cryptographically secured and requires a private key. It cannot be established by unauthorized parties, such as someone who controls the node. The container image contains the SSH host key that can be used for impersonating the host we will connect to. Because this container image is encrypted, and the key to decrypting this image is only provided in measurable ways (e.g. attestation or encrypted initrd), and because the pod/guest memory is protected, even someone who controls the node cannot steal this key.

## Using a pre-provided container image

If you would rather build the image with your own keys, skip to [Building the container image](#building-the-container-image). The [operator](/docs/demos/ccv0-operator-demo) can be used to set up a compatible runtime.

A demo image is provided at [docker.io/katadocker/ccv0-ssh](https://hub.docker.com/r/katadocker/ccv0-ssh).
It is encrypted with [Attestation Agent](https://github.com/confidential-containers/guest-components/tree/main/attestation-agent)'s [offline file system key broker](https://github.com/confidential-containers/guest-components/tree/main/attestation-agent/kbc/src/offline_fs_kbc) and [`aa-offline_fs_kbc-keys.json`](./includes/aa-offline_fs_kbc-keys.json) as its key file.
The private key for establishing an SSH connection to this container is given in [`ccv0-ssh`](./includes/ccv0-ssh).
To use it with SSH, its permissions should be adjusted: `chmod 600 ccv0-ssh`.
The host key fingerprint is `SHA256:wK7uOpqpYQczcgV00fGCh+X97sJL3f6G1Ku4rvlwtR0`.

All keys shown here are for demonstration purposes.
To achieve actually confidential containers, use a hardware trusted execution environment and **do not** reuse these keys.

Continue at [Connecting to the guest](#connecting-to-the-guest).

## Building the container image

The image built should be encrypted.
To receive a decryption key at run time, the Confidential Containers project utilizes the [Attestation Agent](https://github.com/confidential-containers/guest-components/tree/main/attestation-agent).

### Generating SSH keys

```bash
ssh-keygen -t ed25519 -f ccv0-ssh -P "" -C ""
```

generates an SSH key `ccv0-ssh` and the correspondent public key `ccv0-ssh.pub`.

### Building the image

The provided [`Dockerfile`](./includes/Dockerfile) expects `ccv0-sh.pub` to exist.
Using Docker, you can build with

```bash
docker build --progress=plain -t ccv0-ssh .
```

Alternatively, Buildah can be used (`buildah build` or formerly `buildah bud`).
The SSH host key fingerprint is displayed during the build.

## Connecting to the guest

A [Kubernetes YAML file](./includes/k8s-cc-ssh.yaml) specifying the [`kata`](https://github.com/kata-containers/kata-containers) runtime is included.
If you use a [self-built image](#building-the-container-image), you should replace the image specification with the image you built.
The default tag points to an `amd64` image, an `s390x` tag is also available.
With common CNI setups, on the same host, with the service running, you can connect via SSH with

```bash
ssh -i ccv0-ssh root@$(kubectl get service ccv0-ssh -o jsonpath="{.spec.clusterIP}")
```

You will be prompted about whether the host key fingerprint is correct.
This fingerprint should match the one specified above/displayed in the Docker build.

`crictl`-compatible [sandbox](./includes/cri-sandbox-config.yaml) and [container](./includes/cri-container-config.yaml) configurations are also included, which forward the pod SSH port (22) to 2222 on the host (use the `-p` flag in SSH).
