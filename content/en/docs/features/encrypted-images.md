---
title: Encrypted Images
date: 2023-01-24
description: Procedures to encrypt and consume OCI images in a TEE
weight: 22
categories:
- feature 
tags:
- images
---

## Context

A user might want to bundle sensitive data on an OCI (Docker) image. The image layers should only be accessible within a Trusted Execution Environment (TEE).

The project provides the means to encrypt an image with a symmetric key that is released to the TEE only after successful verification and appraisal in a Remote Attestation process. CoCo infrastructure components within the TEE will transparently decrypt the image layers as they are pulled from a registry without exposing the decrypted data outside the boundaries of the TEE.

## Instructions

The following steps require a functional CoCo installation on a Kubernetes cluster. A Key Broker Client (KBC) has to be configured for TEEs to be able to retrieve confidential secrets. We assume `cc_kbc` as a KBC for the CoCo project's Key Broker Service (KBS) in the following instructions, but image encryption should work with other Key Broker implementations in a similar fashion.

{{% alert title="Note" color="primary" %}}
Please ensure you have a recent version of [Skopeo](https://github.com/containers/skopeo/releases) (v1.13.3+) installed locally.
{{% /alert %}}

### Encrypt an image

We extend public image with secret data.

```bash
docker build -t unencrypted - <<EOF
FROM nginx:stable
RUN echo "something confidential" > /secret
EOF
```

The encryption key needs to be a 32 byte sequence and provided to the encryption step as base64-encoded string.

```bash
KEY_FILE="image_key"
head -c 32 /dev/urandom | openssl enc > "$KEY_FILE"
KEY_B64="$(base64 < $KEY_FILE)"
```

The key id is a generic resource descriptor used by the key broker to look up secrets in its storage. For KBS this is composed of three segments: `$repository_name/$resource_type/$resource_tag`

```bash
KEY_PATH="/default/image_key/nginx"
KEY_ID="kbs://${KEY_PATH}"
```

The image encryption logic is bundled and invoked in a container:

```bash
git clone https://github.com/confidential-containers/guest-components.git
cd guest-components
docker build -t coco-keyprovider -f ./attestation-agent/docker/Dockerfile.keyprovider .
```

To access the image from within the container, Skopeo can be used to buffer the image in a directory, which is then made available to the container. Similarly, the resulting encrypted image will be put into an output directory.

```bash
mkdir -p oci/{input,output}
skopeo copy docker-daemon:unencrypted:latest dir:./oci/input
docker run -v "${PWD}/oci:/oci" coco-keyprovider /encrypt.sh -k "$KEY_B64" -i "$KEY_ID" -s dir:/oci/input -d dir:/oci/output
```

We can inspect layer annotations to confirm the expected encryption was applied:

```bash
skopeo inspect dir:./oci/output | jq '.LayersData[0].Annotations["org.opencontainers.image.enc.keys.provider.attestation-agent"] | @base64d | fromjson'
```

Sample output:

```json
{
  "kid": "kbs:///default/image_key/nginx",
  "wrapped_data": "lGaLf2Ge5bwYXHO2g2riJRXyr5a2zrhiXLQnOzZ1LKEQ4ePyE8bWi1GswfBNFkZdd2Abvbvn17XzpOoQETmYPqde0oaYAqVTMcnzTlgdYYzpWZcb3X0ymf9bS0gmMkqO3dPH+Jf4axXuic+ITOKy7MfSVGTLzay6jH/PnSc5TJ2WuUJY2rRtNaTY65kKF2K9YP6mtYBqcHqvPDlFiVNNeTAGv2w1zwaMlgZaSHV+Z1y+xxbOV5e98bxuo6861rMchjCiE7FY37PHD3a5ISogq90=",
  "iv": "Z8bGQL7r6qxSpd4L",
  "wrap_type": "A256GCM"
}
```

Finally, the resulting encrypted image can be provisioned to an image registry.

```bash
ENCRYPTED_IMAGE=some-private.registry.io/coco/nginx:encrypted
skopeo copy dir:./oci/output "docker://${ENCRYPTED_IMAGE}"
```

### Provision image key

Prior to launching a Pod the image key needs to be provisioned to the Key Broker's repository. For a KBS deployment on Kubernetes using the local filesystem as repository storage it would work like this:

```bash
kubectl exec deploy/kbs -- mkdir -p "/opt/confidential-containers/kbs/repository/$(dirname "$KEY_PATH")"
cat "$KEY_FILE" | kubectl exec -i deploy/kbs -- tee "/opt/confidential-containers/kbs/repository/${KEY_PATH}" > /dev/null
```

> **Note:** If you're not using KBS deployment using trustee operator additional namespace may be needed `-n coco-tenant`.

### Launch a Pod

We create a simple deployment using our encrypted image. As the image is being pulled and the CoCo components in the TEE encounter the layer annotations that we saw above, the image key will be retrieved from the Key Broker using the annotated Key ID and the layers will be decrypted transparently and the container should come up.

In this example we default to the Cloud API Adaptor runtime, adjust this depending on the CoCo installation.

```bash
kubectl get runtimeclass -o jsonpath='{.items[].handler}'
```

Sample output:

```bash
kata-remote
```

Export variable:

```bash
CC_RUNTIMECLASS=kata-remote
```

Export KBS address:

```bash
KBS_ADDRESS=scheme://host:port
```

Deploy sample pod:

{{< tabpane text=true right=true persist=header >}}

{{% tab header="Bare metal" %}}

```bash
cat <<EOF> nginx-encrypted.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx-encrypted
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        io.katacontainers.config.hypervisor.kernel_params: "agent.aa_kbc_params=cc_kbc::${KBS_ADDRESS}"
        io.containerd.cri.runtime-handler: ${CC_RUNTIMECLASS}
    spec:
      runtimeClassName: ${CC_RUNTIMECLASS}
      containers:
      - image: ${ENCRYPTED_IMAGE}
        name: nginx
        imagePullPolicy: Always
EOF
kubectl apply -f nginx-encrypted.yaml
```

{{% /tab %}}

{{% tab header="Cloud API Adaptor" %}}

- Create file `$HOME/initdata.toml`
   ```bash
   cat <<EOF> initdata.toml
   algorithm = "sha256"
   version = "0.1.1"
   
   [data]
   "aa.toml" = '''
   [token_configs]
   [token_configs.coco_as]
   url = '${KBS_ADDRESS}'
   
   [token_configs.kbs]
   url = '${KBS_ADDRESS}'
   '''
   
   "cdh.toml"  = '''
   socket = 'unix:///run/confidential-containers/cdh.sock'
   credentials = []
   
   [kbc]
   name = 'cc_kbc'
   url = '${KBS_ADDRESS}'
   '''
   EOF
   ```

- Export variable:

   ```bash
   INIT_DATA_B64=$(cat $HOME/initdata.toml | gzip | base64 -w0)
   ```

- Deploy:
   ```bash
   cat <<EOF> nginx-encrypted.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     labels:
       app: nginx
     name: nginx-encrypted
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
         annotations:
           io.katacontainers.config.runtime.cc_init_data: "${INIT_DATA_B64}"
           io.containerd.cri.runtime-handler: ${CC_RUNTIMECLASS}
       spec:
         runtimeClassName: ${CC_RUNTIMECLASS}
         containers:
         - image: ${ENCRYPTED_IMAGE}
           name: nginx
           imagePullPolicy: Always
   EOF
   kubectl apply -f nginx-encrypted.yaml
   ```

{{% /tab %}}

{{< /tabpane >}}

We can confirm that the image key has been retrieved from KBS.

```bash
kubectl logs -f deploy/kbs | grep "$KEY_PATH"
[2024-01-23T10:24:52Z INFO  actix_web::middleware::logger] 10.244.0.1 "GET /kbs/v0/resource/default/image_key/nginx HTTP/1.1" 200 530 "-" "attestation-agent-kbs-client/0.1.0" 0.000670
```

> **Note:** If you're not using KBS deployment using trustee operator additional namespace may be needed `-n coco-tenant`.

## Debugging

The encrypted image feature relies on a cascade of other features as building blocks:

- Image decryption is built on secret retrieval from KBS
- Secret retrieval is built on remote attestation
- Remote attestation requires retrieval of hardware evidence from the TEE

All of the above have to work in order to decrypt an encrypted image.

### 0. Launching with an unencrypted image

Launch the same image with unencrypted layers _from the same registry_ to verify that the image itself is not an issue.

### 1. Retrieve hardware evidence from TEE

Launch an unencrypted `library/nginx` deployment named `nginx` with a CoCo runtime class. Issue `kubectl exec deploy/nginx -- curl http://127.0.0.1:8006/aa/evidence\?runtime_data\=xxxx`. This should produce a real hardware evidence rendered as JSON. Something like `{"svn":"1","report_data":"eHh4eA=="}` is not real hardware evidence, it's a dummy response from the sample attester. Real TEE evidence should be more verbose and contain certificates and measurements.

The reason for not producing real evidence could be a wrong build of Attestation Agent for the TEE that you are attempting to use, or the Confidential VM not exposing the expected interfaces.

Note: In some configurations of the CoCo image the facility to retrieve evidence is disabled by default. For bare-metal CoCo images you can enable it by setting `agent.guest_components_rest_api=all` on the kernel cmdline (see [here](https://github.com/kata-containers/kata-containers/blob/main/src/agent/README.md#agent-options)).

### 2. Perform remote attestation

Run `kubectl exec deploy/nginx -- curl http://127.0.0.1:8006/aa/token\?token_type\=kbs`. This should produce an attestation token. If you don't receive a token but an error you should inspect your Trustee KBS/AS logs and see whether there was a connection attempt for the CVM and potentially a reason why remote attestation failed. 

You can set `RUST_LOG=debug` as environment variable the Trustee deployment to receive more verbose logs. If the evidence is being sent to KBS, the issue is most likely resolvable on the KBS side and possibly related to the evidence not meeting the expectations for a key release in KBS/AS policies or the requested resource not being available in the KBS.

If you don't see an attestation attempt in the log of KBS there might be problems with network connectivity between the Confidential VM and the KBS. Note that the Pod and the Guest Components on the CVM might not share the same network namespace. A Pod might be able to reach KBS, but the Attestation Agent on the Confidential VM might not. If you are using HTTPS to reach the KBS, there might be a problem with the certificate provided to the Confidential VM (e.g via Initdata).

### 3. Retrieve decryption key from KBS

If you have successfully retrieved a token attempt to fetch the symmetric key for the encrypted image, manually and using the image's key id (kid): `kubectl exec deploy/nginx -- curl http://127.0.0.1:8006/cdh/resource/default/images/my-key`.

You can find out the kid for a given encrypted image in the a query like this:

```bash
$ ANNOTATION="org.opencontainers.image.enc.keys.provider.attestation-agent"
$ skopeo inspect docker://ghcr.io/mkulke/nginx-encrypted@sha256:5a81641ff9363a63c3f0a1417d29b527ff6e155206a720239360cc6c0722696e \
    | jq --arg ann "$ANNOTATION" -r '.LayersData[0].Annotations[$ann] | @base64d | fromjson | .kid' 
kbs:///default/image_key/nginx
```

If the key can be retrieved successfully, verify that the size is exactly 32 bytes and matches the key you used for encrypting the image.

### 4. Other (desperate) measures

If pulling an encrypted image still doesn't work after successful retrieval of its encryption key, you might want to purge the node(s). There are bugs in the containerd remote snapshotter implementation that might taint the node with manifests and layers that will interfere with image pulling. The node should be discarded and reinstalled to rule that out.

It might also help to set `annotations.io.containerd.cri.runtime-handler: $your-runtime-class` in the pod spec, in addition to the `runtimeClassName` field and `ImagePullPolicy: Always` to ensure that the image is always pulled from the registry.
