---
title: Authenticated Registries 
date: 2024-12-06
description: Use private OCI registries
categories:
- feature 
tags:
- authenticated registries
---

# Context

A user might want to use container images from private OCI registries, hence requiring authentication. The project provides the means to pull protected
images from authenticated registry.

> **Important**: ideally the authentication credentials should only be accessible from within a Trusted Execution Environment, however, due some limitations
on the architecture of components used by CoCo, the credentials need to be exposed to the host, thus **registry authentication is not currently a confidential feature**. The community has worked to remediate that limitation and, in meanwhile, we recommend the use of [encrypted images](./encrypted-images.md) as a mitigation.

# Instructions

The following steps require a functional CoCo installation on a Kubernetes cluster. A Key Broker Client (KBC) has to be configured for TEEs to be able to retrieve confidential secrets. We assume `cc_kbc` as a KBC for the CoCo project's Key Broker Service (KBS) in the following instructions, but authenticated registries should work with other Key Broker implementations in a similar fashion.

## Create registry authentication file

The registry authentication file should have the [`containers-auth.json` format](https://github.com/containers/image/blob/main/docs/containers-auth.json.5.md#format), with exception of credential helpers (`credHelpers`) that aren't supported. Also it's not supported glob URLs nor prefix-matched paths as in
Kubernetes [interpretation of `config.json`](https://kubernetes.io/docs/concepts/containers/images/#config-json).

Create the registry authentication file (e.g `containers-auth.json`) like this:

```bash
export AUTHENTICATED_IMAGE="my-registry.local/repository/image:latest"
export AUTHENTICATED_IMAGE_NAMESPACE="$(echo "$AUTHENTICATED_IMAGE" | cut -d':' -f1)"
export AUTHENTICATED_IMAGE_USER="MyRegistryUser"
export AUTHENTICATED_IMAGE_PASSWORD="MyRegistryPassword"
cat <<EOF>> containers-auth.json
{
	"auths": {
		"${AUTHENTICATED_IMAGE_NAMESPACE}": {
			"auth": "$(echo ${AUTHENTICATED_IMAGE_USER}:${AUTHENTICATED_IMAGE_PASSWORD} | base64 -w 0)"
		}
	}
}
EOF
```

Where:
 * `AUTHENTICATED_IMAGE` is the full-qualified image name
 * `AUTHENTICATED_IMAGE_NAMESPACE` is the image name without the tag
 * `AUTHENTICATED_IMAGE_USER` and `AUTHENTICATED_IMAGE_PASSWORD` are the registry credentials user and password, respectively
 * `auth`'s value is the colon-separated user and password (`user:password`) credentials string encoded in base64

## Provision the registry authentication file

Prior to launching a Pod the registry authentication file needs to be provisioned to the Key Broker's repository. For a KBS deployment on Kubernetes using the local filesystem as repository storage it would work like this:

```bash
export KEY_PATH="default/containers/auth"
kubectl exec deploy/kbs -c kbs -n coco-tenant -- mkdir -p "/opt/confidential-containers/kbs/repository/$(dirname "$KEY_PATH")"
cat containers-auth.json | kubectl exec -i deploy/kbs -c kbs -n coco-tenant -- tee "/opt/confidential-containers/kbs/repository/${KEY_PATH}" > /dev/null
```

The CoCo infrastructure components need to cooperate with containerd and nydus-snapshotter to pull the container image from TEE. Currently
the nydus-snapshotter needs to fetch the image's metadata from registry, then authentication credentials are read from a Kubernetes secret
of `docker-registry` type. So it should be created a secret like this:

```bash
export SECRET_NAME="cococred"
kubectl create secret docker-registry "${SECRET_NAME}" --docker-server="https://${AUTHENTICATED_IMAGE_NAMESPACE}" \
    --docker-username="${AUTHENTICATED_IMAGE_USER}" --docker-password="${AUTHENTICATED_IMAGE_PASSWORD}"
```

Where:
 * `SECRET_NAME` is any secret name

## Launch a Pod

Create the pod yaml (e.g. pod-image-auth.yaml) like below and apply it:

```bash
export KBS_ADDRESS="172.18.0.3:31731"
export RUNTIMECLASS="kata-qemu-coco-dev"
cat <<EOF>> pod-image-auth.yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-auth-feat
  annotations:
    io.containerd.cri.runtime-handler: ${RUNTIMECLASS}
    io.katacontainers.config.hypervisor.kernel_params: ' agent.image_registry_auth=kbs:///${KEY_PATH} agent.guest_components_rest_api=resource agent.aa_kbc_params=cc_kbc::http://${KBS_ADDRESS}'
spec:
  runtimeClassName: ${RUNTIMECLASS}
  containers:
    - name: test-container
      image: ${AUTHENTICATED_IMAGE}
      imagePullPolicy: Always
      command:
        - sleep
        - infinity
  imagePullSecrets:
    - name: ${SECRET_NAME}
EOF
```

Where:

 * `KBS_ADDRESS` is the `host:port` address of KBS
 * `RUNTIMECLASS` is any of available CoCo runtimeclasses (e.g. `kata-qemu-tdx`, `kata-qemu-snp`). For this example, `kata-qemu-coco-dev` allows to create CoCo pod on systems without confidential hardware. It should be replaced with a class matching the TEE in use.

What distinguish the pod specification for authenticated registry from a regular CoCo pod is:

 * the `agent.image_registry_auth` property in `io.katacontainers.config.hypervisor.kernel_params` annotation indicates the location of the registry authentication file as a resource in the KBS
 * the `imagePullSecrets` as required by nydus-snapshotter

Check the pod gets *Running*:

```bash
kubectl get -f pod-image-auth.yaml
NAME              READY   STATUS    RESTARTS   AGE
image-auth-feat   1/1     Running   0          2m52s
```