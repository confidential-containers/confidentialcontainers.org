---
title: Sealed Secrets 
date: 2023-01-24
description: Generate and deploy protected Kubernetes secrets
categories:
- feature 
tags:
- secrets 
---

{{% alert title="Note" color="primary" %}}
Sealed Secrets depend on attestation.
[Configure attestation](../attestation) before using sealed secrets.
{{% /alert %}}

Sealed secrets allow confidential information to be stored in the untrusted control plane.
Like normal Kubernetes secrets, sealed secrets are orchestrated by the control plane
and are transparently provisioned to your workload as environment variables or volumes.

## Basic Usage

Here's how you create a vault secret.
There are also envelope secrets, which are described later.
Vault secrets are a pointer to resource stored in a KBS,
while envelope secrets are wrapped secrets that are unwrapped with a KMS.

### Creating a sealed secret

There is a helper tool for sealed secrets in the Guest Components repository.

Clone the repository.
```bash
git clone https://github.com/confidential-containers/guest-components.git
```

Inside the `guest-components` directory, you can build and run the tool with `Cargo`.
```bash
cargo run -p confidential-data-hub --bin secret
```

With the tool you can create a secret.
```bash
cargo run -p confidential-data-hub --bin secret seal vault --resource-uri kbs:///your/secret/here --provider kbs
```

A vault secret is fulfilled by retrieving a secret from a KBS inside the guest.
The locator of your secret is specified by `resource-uri`.

This command should return a base64 string which you will use in the next step.

{{% alert title="Note" color="primary" %}}
For vault secrets, the secret-cli tool does not upload your resource to the KBS
automatically.
In addition to generating the secret string, you must also upload the resource
to your KBS.
{{% /alert %}}

### Adding a sealed secret to Kubernetes

Create a secret from your secret string using `kubectl`.
```bash
kubectl create secret generic sealed-secret --from-literal='secret=sealed.fakejwsheader.ewogICAgInZlcnNpb24iOiAiMC4xLjAiLAogICAgInR5cGUiOiAidmF1bHQiLAogICAgIm5hbWUiOiAia2JzOi8vL2RlZmF1bHQvc2VhbGVkLXNlY3JldC90ZXN0IiwKICAgICJwcm92aWRlciI6ICJrYnMiLAogICAgInByb3ZpZGVyX3NldHRpbmdzIjoge30sCiAgICAiYW5ub3RhdGlvbnMiOiB7fQp9Cg==.fakesignature'
```

{{% alert title="Note" color="primary" %}}
Sealed secrets do not currently support integrity protection.
This will be added in the future, but for now a fake signature
and signature header are included within the secret.
{{% /alert %}}

When using `--from-literal` you provide a mapping of secret keys and values. 
The secret value should be the string generated in the previous step.
The secret key can be whatever you want, but make sure to use the same one in future steps.
This is separate from the name of the secret.

### Deploying a sealed secret to a confidential workload

You can add your sealed secret to a workload yaml file.

You can expose your sealed secret as an environment variable.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sealed-secret-pod
spec:
  runtimeClassName: kata-qemu-coco-dev
  containers:
  - name: busybox
    image: quay.io/prometheus/busybox:latest
    imagePullPolicy: Always
    command: ["echo", "$PROTECTED_SECRET"]
    env:
    - name: PROTECTED_SECRET
      valueFrom:
        secretKeyRef:
          name: sealed-secret
          key: secret
```

You can also expose your secret as a volume.
```yaml 
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod-cc
spec:
  runtimeClassName: kata
  containers:
  - name: busybox
    image: quay.io/prometheus/busybox:latest
    imagePullPolicy: Always
    command: ["cat", "/sealed/secret-value/secret"]
    volumeMounts:
        - name: sealed-secret-volume
          mountPath: "/sealed/secret-value"
  volumes:
    - name: sealed-secret-volume
      secret:
        secretName: sealed-secret
```
{{% alert title="Note" color="primary" %}}
Currently sealed secret volumes must be mounted
in the `/sealed` directory.
{{% /alert %}}

## Advanced

### Envelope Secrets

You can also create envelope secrets.
With envelope secrets, the secret itself is included in the secret
(unlike a vault secret, which is just a pointer to a secret).
In an envelope secret, the secret value is wrapped and can be unwrapped
by a KMS.
This allows us to support models where the key for unwrapping secrets
never leaves the KMS.
It also decouples the secret from the KBS.

We currently support two KMSes for envelope secrets.
See specific instructions for [aliyun kms](https://github.com/confidential-containers/guest-components/blob/main/confidential-data-hub/docs/kms-providers/alibaba.md)
and [eHSM](https://github.com/confidential-containers/guest-components/blob/main/confidential-data-hub/docs/kms-providers/ehsm-kms.md).
