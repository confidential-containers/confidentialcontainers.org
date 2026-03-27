---
title: Signed Images
date: 2023-01-24
description: Procedures to generate and deploy signed OCI images with CoCo
weight: 21
categories:
  - feature
tags:
  - images
---

## Overview

[Encrypted images](/docs/features/encrypted-images/) provide confidentiality,
but they do not provide _authenticity_ or _integrity_. Image signatures provide
this additional property, preventing certain types of image tampering,
[for example](https://docs.sigstore.dev/about/overview/#why-cryptographic-signing).

In this brief guide, we show two tools that can be used to sign container images:
[cosign](https://github.com/sigstore/cosign) and
[skopeo](https://github.com/containers/skopeo). The skopeo tool can be
used to create both cosign signatures or "simple signatures" (which leverage
gpg keys). For our purposes, our skopeo examples will use the simple signing approach.

In any case, the general approach is to

1. Create keys for signing,
2. Sign a newly tagged image, and
3. Update the [KBS](/docs/attestation/key-broker-service) with the public
   signature key and a security policy.

## Creating an Image

### Creating a Key Pair

Create a key pair using one of two approaches: `cosign` or simple signing with gpg.

{{< tabpane text=true right=true persist=header >}}

{{% tab header="cosign" %}}

To generate a public/private key pair with `cosign`, set `COSIGN_PASSWORD`
and run `generate-key-pair`:

```bash
COSIGN_PASSWORD=just1testing2password3 cosign generate-key-pair
```

This will create the private and public keys: `cosign.key` and `cosign.pub`.

{{% /tab %}}

{{% tab header="Simple Signing - gpg" %}}

`skopeo` depends on gpg for key management.
To generate a key pair with gpg using the default options, run:

```bash
gpg --full-generate-key
```

You will be prompted for several values. For testing, you can use:

```text
GitHub Runner
git@runner.com
just1testing2password3
```

Then export the key material. The `--export-secret-key` option is sufficient
to export both the private and public keys. For example:

```bash
gpg --export-secret-key F63DB2A1AB7C7F195F698C9ED9582CADF7FBCC5D > github-runner.keys
```

You can later import the keys in a CI system by using `--batch` to avoid
interactive prompts:

```bash
gpg --batch --import ./github-runner.keys
```

When automating CI or test workflows, you can place the key password in a
plain-text file when that is acceptable for your environment:

```bash
echo just1testing2password3 > git-runner-password.txt
```

{{% /tab %}}

{{< /tabpane >}}

### Signing the Image

Sign the image using one of two approaches: `cosign` or simple signing with `skopeo`.

{{< tabpane text=true right=true persist=header >}}

{{% tab header="cosign" %}}

In this example, we use a sample minimal `Dockerfile` to build an image that will be signed.

Create `Dockerfile`:

```bash
cat <<EOF > Dockerfile
FROM nginx:1.27-alpine

EXPOSE 80
EOF
```

The workflow is to build the image, push it to ghcr, and then sign it.

Make sure you are authenticated to ghcr first, for example with `docker login`.

Perform the following steps to build, push, and sign the image:

1. Build the image

    ```bash
    COCO_PKG=confidential-containers/test-container
    docker build \
      -t ghcr.io/${COCO_PKG}:cosign-sig \
      -f Dockerfile \
      .
    ```
   
2. Push the image to ghcr

    ```bash
    docker push ghcr.io/${COCO_PKG}:cosign-sig
    ```

   After pushing the image, note the image digest shown in the output.
   You will use it in the signing command. For example:

    ```text
    cosign-sig: digest: sha256:<IMAGE_DIGEST> size: 1989
    ```

3. Check your `cosign` version:

    ```bash
    cosign version
    ```

4. Use the signing command that matches your installed version.

##### cosign `v3.0.x`

```bash
cosign sign --new-bundle-format=false \
  --use-signing-config=false \
  --key ./cosign.key \
  ghcr.io/${COCO_PKG}@sha256:<IMAGE_DIGEST>
```

##### cosign `>= v2.2.0` and `< v3.0`

```bash
cosign sign --key ./cosign.key ghcr.io/${COCO_PKG}@sha256:<IMAGE_DIGEST>
```

{{% alert title="Note" color="primary" %}}
`cosign` versions `v2.0.x` and `v2.1.x` use a different private key format from `>= v2.2.0`.
{{% /alert %}}

{{% /tab %}}

{{% tab header="Simple Signing - skopeo" %}}

Ensure that you have a gpg key owned by the user signing the image. See the
previous subsection for instructions on generating and importing gpg keys.

The following example signs a local image named
`confidential-containers/test-container`. It uses the `unsigned` tag and, as
part of the signing flow, creates a new `simple-signed` tag. In this example,
the resulting image is pushed to ghcr, which requires `docker login` first:

```bash
COCO_PKG=confidential-containers/test-container
skopeo \
  copy \
  --debug \
  --insecure-policy \
  --sign-by git@runner.com \
  --sign-passphrase-file ./git-runner-password.txt \
  docker-daemon:ghcr.io/${COCO_PKG}:unsigned \
  docker://ghcr.io/${COCO_PKG}:simple-signed
```

{{% /tab %}}

{{< /tabpane >}}

## Running an Image

Running a workload with a signed image is very similar to running a workload with an unsigned image.
The main difference is that, for a signed image, you must provide the [KBS](/docs/attestation/architecture/)
with the public key and a security policy.

The security policy tells KBS which image is signed, which signature type is
used, and where to find the public key that should be used for verification.
After that, you can run the workload as usual, for example with `kubectl apply`.

### Setting the Security Policy for Signed Images

Register the public key in KBS storage. For example:

{{< tabpane text=true right=true persist=header >}}

{{% tab header="Trustee Operator" %}}

Export the `KbsConfig` custom resource name:

```bash
export CR_NAME=$(kubectl get kbsconfig -n trustee-operator-system -o=jsonpath='{.items[0].metadata.name}')
```

Create a Secret containing the public key:

```bash
kubectl create secret generic sig-public-key \
  -n trustee-operator-system \
  --from-file=test=./cosign.pub
```

Patch the `KbsConfig` custom resource to add the public key Secret:

```bash
kubectl patch KbsConfig -n trustee-operator-system $CR_NAME \
  --type=json \
  -p='[{"op":"add", "path":"/spec/kbsSecretResources/-", "value":"sig-public-key"}]'
```

{{% /tab %}}

{{% tab header="kbs-client tool" %}}

Run the following command to add the public key to KBS storage:

```bash
./kbs-client --url <SCHEME>://<HOST>:<PORT> config \
  --auth-private-key private.key \
  set-resource \
  --resource-file cosign.pub \
  --path default/sig-public-key/test
```

{{% /tab %}}

{{< /tabpane >}}

Create an image pull validation policy file.
For example, create `security-policy.json` with the following contents:

```json
{
  "default": [
    {
      "type": "reject"
    }
  ],
  "transports": {
    "<transport>": {
      "<registry>/<image>": [
        {
          "type": "sigstoreSigned",
          "keyPath": "kbs:///default/<type>/<tag>"
        }
      ]
    }
  }
}
```

By default, the policy rejects all images and all signatures. The transports section specifies which images the policy
explicitly approves and verifies through their signatures.

Replace placeholders in the policy file with the appropriate values:

- `<transport>` - Specify the image repository for transport, for example, `docker`. More information can be found
  in [containers-transports 5](https://github.com/containers/image/blob/main/docs/containers-transports.5.md).
- `<registry>/<image>` - Specify the container registry and image, for example,
  `ghcr.io/confidential-containers/test-container`.
- `<type>/<tag>` - Specify the type and tag of the container image signature verification secret that you created, for
  example, `sig-public-key/test`.

Finally, register the image pull validation policy file in KBS storage:

{{< tabpane text=true right=true persist=header >}}

{{% tab header="Trustee Operator" %}}

Export the `KbsConfig` custom resource name:

```bash
export CR_NAME=$(kubectl get kbsconfig -n trustee-operator-system -o=jsonpath='{.items[0].metadata.name}')
```

Create a Secret containing the security policy:

```bash
kubectl create secret generic security-policy \
  -n trustee-operator-system \
  --from-file=test=./security-policy.json
```

Patch the `KbsConfig` custom resource to add the security policy Secret:

```bash
kubectl patch KbsConfig -n trustee-operator-system $CR_NAME \
  --type=json \
  -p='[{"op":"add", "path":"/spec/kbsSecretResources/-", "value":"security-policy"}]'
```

{{% /tab %}}

{{% tab header="kbs-client tool" %}}

Run the following command to add the security policy to KBS storage:

```bash
./kbs-client --url <SCHEME>://<HOST>:<PORT> config \
  --auth-private-key private.key \
  set-resource \
  --resource-file ./security-policy.json \
  --path default/security-policy/test
```

{{% /tab %}}

{{< /tabpane >}}

### Enable Signature Verification

To enforce signature verification for a Pod, you must set the appropriate kernel parameters or init data in the pod
annotation.

{{< tabpane text=true right=true persist=header >}}

{{% tab header="kernel params" %}}

Set the following kernel parameters in the `io.katacontainers.config.hypervisor.kernel_params` pod annotation:

```text
agent.image_policy_file=kbs:///default/<SECRET_POLICY_NAME>/<KEY> agent.enable_signature_verification=true agent.aa_kbc_params=cc_kbc::<SCHEME>://<HOST>:<PORT>
```

- `agent.image_policy_file` points to the security policy file registered in KBS storage. In this example,
  `SECRET_POLICY_NAME` = `security-policy` and `KEY` = `test`.
- `agent.enable_signature_verification` enables signature verification.
- `agent.aa_kbc_params` points to the KBS service. Replace `<SCHEME>`, `<HOST>`, and `<PORT>` with the values for your
  environment.

{{% /tab %}}

{{% tab header="init-data" %}}

Run the following commands to prepare the init data file with the appropriate KBS configuration and security policy:

- Export environment variables:

  ```bash
  export KBS_ADDRESS=scheme://host:port
  export SECRET_POLICY_NAME=security-policy
  export SECRET_POLICY_KEY=test
  ```

- Create file `$HOME/initdata.toml`
   ```bash
   cat <<EOF> initdata.toml
   algorithm = "sha256"
   version = "0.1.0"
   
   [data]
   "aa.toml" = '''
   [token_configs]
   [token_configs.coco_as]
   url = '${KBS_ADDRESS}'
   
   [token_configs.kbs]
   url = '${KBS_ADDRESS}'
   '''
   
   "cdh.toml" = '''
   socket = 'unix:///run/confidential-containers/cdh.sock'
   credentials = []
   
   [kbc]
   name = 'cc_kbc'
   url = '${KBS_ADDRESS}'
   
   [image]
   image_security_policy_uri = 'kbs:///default/${SECRET_POLICY_NAME}/${SECRET_POLICY_KEY}'
   '''
   EOF
   ```

  The most important fields in the `[image]` section are:

  - `image_security_policy_uri` - Points to the image security policy that `image-rs` uses when pulling images.
    Commonly a KBS URI such as `kbs:///default/...`, but it can also point to a local file, for example `file:///etc/image-policy.json`. 
    If this field is not set, no image security policy is applied and pulled images are not validated.
  - `image_security_policy` - Inline alternative that lets you provide the policy content directly as a string instead of referencing it through a URI. 
    If both `image_security_policy_uri` and `image_security_policy` are set, `image_security_policy_uri` takes precedence.

- Encode the init data file in base64:

  ```bash
  export INIT_DATA=$(cat $HOME/initdata.toml | gzip | base64 -w 0)
  ```

Set the output from above command in the `io.katacontainers.config.hypervisor.cc_init_data` pod annotation:

```text
io.katacontainers.config.hypervisor.cc_init_data = ${INIT_DATA}
```

{{% /tab %}}

{{< /tabpane >}}

### Run a Signed Workload

Create a Pod that uses the signed image.

Replace `<SCHEME>`, `<KBS_HOST>`, `<KBS_PORT>`, and `runtimeClassName` with
the values appropriate for your environment:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test-container
  name: test-container
  annotations:
    io.katacontainers.config.hypervisor.kernel_params: agent.aa_kbc_params=cc_kbc::<SCHEME>://<KBS_HOST>:<KBS_PORT> agent.image_policy_file=kbs:///default/security-policy/test agent.enable_signature_verification=true
spec:
  containers:
    - name: test-container
      image: ghcr.io/confidential-containers/test-container:cosign-sig
  dnsPolicy: ClusterFirst
  runtimeClassName: <RUNTIME_CLASS>
EOF
```

Verify that the Pod is running:

```console
$ kubectl get pod test-container -o jsonpath='{.status.phase}{"\n"}'
Running
```

## Troubleshooting

If image signature verification fails, you may see an error similar to the following in the Pod log:

```text
Image Pull error: Failed to pull image [IMAGE] from all mirror/mapping locations or original location: \
image: [IMAGE], error: Image policy rejected: Denied by policy: rejected by `sigstoreSigned` rule
```

In that case, check the following:

- Ensure that the public key is correctly registered in KBS storage and that the key path in the security policy file is
  correct.
- Ensure that the security policy file is correctly registered in KBS storage and that the path in the pod annotation is
  correct.
- Ensure that the image is correctly signed and that the signature is valid.

## See Also

### Cosign GitHub Integration

A good tutorial for `cosign` and GitHub integration is available
[here](https://dev.to/n3wt0n/sign-your-container-images-with-cosign-github-actions-and-github-container-registry-3mni).
The approach is automated and targets real-world usage.
For example, the following key generation step automatically uploads the
public key, private key, and password secret to the GitHub repository:

```bash
GITHUB_TOKEN=<GITHUB_TOKEN> \
COSIGN_PASSWORD=just1testing2password3 \
cosign generate-key-pair github://<github_username>/<github_repo>
```
