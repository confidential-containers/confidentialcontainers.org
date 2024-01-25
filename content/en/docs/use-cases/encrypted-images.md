---
title: Encrypted images
date: 2023-01-24
description: Procedures to encrypt and consume OCI images in a TEE
categories:
- use case
tags:
- coco-keyprovider
- images
- kbs
---

# Context

A user might want to bundle sensitive data on an OCI (Docker) image. The image layers should only be accessible within a Trusted Execution Environment (TEE).

The project provides the means to encrypt an image with a symmetric key that is released to the TEE only after successful verification and appraisal in a Remote Attestation process. CoCo infrastructure components within the TEE will transparently decrypt the image layers as they are pulled from a registry without exposing the decrypted data outside the boundaries of the TEE.

# Instructions

The following steps require a functional CoCo installation on a Kubernetes cluster. A Key Broker Client (KBC) has to be configured for TEEs to be able to retrieve confidential secrets. We assume `cc_kbc` as a KBC for the CoCo project's Key Broker Service (KBS) in the following instructions, but image encryption should work with other Key Broker implementations in a similar fashion.

Please ensure you have a recent version of [Skopeo](https://github.com/containers/skopeo/releases) (v1.14.2+) installed locally.

## Encrypt an image

We extend public image with secret data.

```bash
docker build -t unencrypted - <<EOF
FROM nginx:stable
RUN echo "something confidential" > /secret
EOF
```

The encryption key needs to be a 32 byte sequence and provided to the encryption step as base64-encoded string.

```bash
head -c 32 /dev/urandom | openssl enc > image_key
KEY_B64="$(base64 < image_key)"
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
{
  "kid": "kbs:///default/image_key/one",
  "wrapped_data": "lGaLf2Ge5bwYXHO2g2riJRXyr5a2zrhiXLQnOzZ1LKEQ4ePyE8bWi1GswfBNFkZdd2Abvbvn17XzpOoQETmYPqde0oaYAqVTMcnzTlgdYYzpWZcb3X0ymf9bS0gmMkqO3dPH+Jf4axXuic+ITOKy7MfSVGTLzay6jH/PnSc5TJ2WuUJY2rRtNaTY65kKF2K9YP6mtYBqcHqvPDlFiVNNeTAGv2w1zwaMlgZaSHV+Z1y+xxbOV5e98bxuo6861rMchjCiE7FY37PHD3a5ISogq90=",
  "iv": "Z8bGQL7r6qxSpd4L",
  "wrap_type": "A256GCM"
}
```

Finally the resulting encrypted image can be provisioned to an image registry.

```bash
ENCRYPTED_IMAGE=some-private.registry.io/coco/nginx:encrypted
skopeo copy dir:./oci/output "docker://${ENCRYPTED_IMAGE}"
```

## Provision image key

Prior to launching a Pod the image key needs to be provisioned to the Key Broker's repository. For a KBS deployment on Kubernetes using the local filesystem as repository storage it would work like this:

```bash
kubectl exec deploy/kbs -- mkdir -p "/opt/confidential-containers/kbs/repository/$(dirname "$KEY_PATH")"
cat "$KEY_FILE" | kubectl exec -i deploy/kbs -- tee "/opt/confidential-containers/kbs/repository/${KEY_PATH}" > /dev/null
```

## Launch a Pod

In this example we default to the Cloud API Adaptor runtime, adjust this depending on the CoCo installation.

```bash
kubectl get runtimeclass -o jsonpath='{.items[].handler}'
kata-remote
CC_RUNTIMECLASS=kata-remote
```

We create a simple deployment using our encrypted image. As the image is being pulled and the CoCo components in the TEE encounter the layer annotations that we saw above, the image key will be retrieved from the Key Broker using the annotated Key ID and the layers will be decrypted transparently and the container should come up.
  
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
        io.containerd.cri.runtime-handler: ${CC_RUNTIMECLASS}
    spec:
      runtimeClassName: ${CC_RUNTIMECLASS}
      containers:
      - image: ${ENCRYPTED_IMAGE}
        name: nginx
EOF
kubectl apply -f nginx-encrypted.yaml
``` 

We can confirm that the image key has been retrieved from KBS.

```bash
kubectl logs -f deploy/kbs | grep "$KEY_PATH"
[2024-01-23T10:24:52Z INFO  actix_web::middleware::logger] 10.244.0.1 "GET /kbs/v0/resource/default/image_key/nginx HTTP/1.1" 200 530 "-" "attestation-agent-kbs-client/0.1.0" 0.000670
```
