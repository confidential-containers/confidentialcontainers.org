---
title: Signed Images
date: 2023-01-24
description: Procedures to generate and deploy signed OCI images with CoCo
categories:
- feature 
tags:
- images
---



# Overview
[Encrypted images](/docs/features/encrypted-images/) provide confidentiality,
but they do not provide _authenticity_. Image signatures provide this additional
property, preventing certain types of image tampering,
[for example](https://docs.sigstore.dev/about/overview/#why-cryptographic-signing).

In this brief guide, we show two tools that can be used to sign container images:
[cosign](https://github.com/sigstore/cosign) and
[skopeo](https://github.com/containers/skopeo). The skopeo approach is often
called "simple signing" in the CoCo project.  In both cases, the general
approach is to (1) create keys for signing, (2) sign a newly tagged image, and
(3) update the [KBS](/docs/attestation/key-broker-service) with the public
signature key and a security policy.


# Creating Keys

## cosign
To generate a public-private keypair with cosign, provide your
`COSIGN_PASSWORD` and use the `generate-key-pair` action:
```shell
$ COSIGN_PASSWORD=just1testing2password3 cosign generate-key-pair
```

## Simple Signing - gpg
skopeo depends on gpg for a keypair.
To generate a keypair with gpg using default options, use `--full-generate-key`:
```shell
$ gpg --full-generate-key
```

There are several prompts. A user for test purposes could be:
```
Github Runner
git@runner.com
just1testing2password3
```

Then export it. The `--export-secret-key` option is sufficient for exporting
both the secret and public key. Example command:
```shell
$ gpg --export-secret-key F63DB2A1AB7C7F195F698C9ED9582CADF7FBCC5D > github-runner.keys
```

The keys can later be imported by gpg in a CI system using `--batch` to avoid
typing the password:
```shell
$ gpg --batch --import ./github-runner.keys
```

When automating CI or test worfklows, you can place the password for the key in
a plaintext file (when it is safe to do so):
```shell
echo just1testing2password3 > git-runner-password.txt
```


# Signing the Image
## cosign

Use the private key to sign an image. In this example, the image is on ghcr,
which requires `docker login`:
```shell
COCO_PKG=confidential-containers/test-container
cosign sign --key ./cosign.key ghcr.io/${COCO_PKG}:cosign-signed
```

## Simple Signing - skopeo
Ensure you have a gpg key owned by the user signing the image. See the previous
subsection for generating and importing gpg keys.

Sign the image. For example, the following command uses an insecure-policy
to sign a local image called `confidential-containers/test-container`. It uses
the `unsigned` tag, and in the process of signing it, creates a new
`simple-signed` tag.
In this example, the resulting image is pushed to ghcr, which requires `docker
login`:
```shell
COCO_PKG=confidential-containers/test-container
skopeo \
  copy \
  --debug \
  --insecure-policy \
  --sign-by git@runner.com \
  --sign-passphrase-file ./git-runner-password.txt \
  docker-daemon:ghcr.io/${COCO_PKG}:unsiged \
  docker://ghcr.io/${COCO_PKG}:simple-signed
```




# Setting the Security Policy to Use the Signed Image
Register the public key to KBS storage. For example:
```shell
mkdir -p ${KBS_DIR_PATH}/data/kbs-storage/default/cosign-key \
  && cp cosign.pub ${KBS_DIR_PATH}/data/kbs-storage/default/cosign-key/1
```

Edit an image pulling validation policy file.
Here is a sample policy file `security-policy.json`:
```json
{
    "default": [{"type": "reject"}],
    "transports": {
        "docker": {
            "[REGISTRY_URL]": [
                {
                    "type": "sigstoreSigned",
                    "keyPath": "kbs:///default/cosign-key/1"
                }
            ]
        }
    }
}
```
Be sure to replace `[REGISTRY_URL]` with the desired registry URL of the
encrypted image.

Lastly, register the image pulling validation policy file with KBS storage:
```shell
mkdir -p ${KBS_DIR_PATH}/data/kbs-storage/default/security-policy
cp security-policy.json ${KBS_DIR_PATH}/data/kbs-storage/default/security-policy/test
```




# See Also
## Cosign-GitHub Integration
A good tutorial for cosign and github integration is
[here](https://dev.to/n3wt0n/sign-your-container-images-with-cosign-github-actions-and-github-container-registry-3mni).
The approach is automated and targets real-world usage.
For example, this key-generation step automatically
uploads the public key, private key, and key secret to the github repo:
```
$ GITHUB_TOKEN=ghp_... \
COSIGN_PASSWORD=just1testing2password3 \
cosign generate-key-pair github://portersrc/infra
```