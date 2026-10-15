---
title: Verify and clean up a CAA deployment
linkTitle: Verify and clean up
description: Shared verification, sample workload, and cleanup steps for Cloud API Adaptor deployments
categories:
  - cloud-providers
tags:
  - helm
  - caa
weight: 2
---

Use these steps after installing the Cloud API Adaptor (CAA) with any cloud-provider guide.

## Verify the installation

Confirm that the `kata-remote` runtime class exists:

```bash
kubectl get runtimeclass
```

A successful deployment returns output similar to:

```console
$ kubectl get runtimeclass
NAME          HANDLER       AGE
kata-remote   kata-remote   7m18s
```

## Run a sample workload

Create a nginx deployment that uses the `kata-remote` runtime class:

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      runtimeClassName: kata-remote
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
EOF
```

Wait for the pod to become ready:

```bash
kubectl wait --for=condition=Ready pod -l app=nginx --timeout=10m
kubectl get pods -l app=nginx -o wide
```

Return to your provider guide for the command that confirms creation of the corresponding PodVM in the cloud console or API.

If the pod does not start, see the [troubleshooting guide]({{< relref "../troubleshooting/_index.md" >}}).

## Remove the sample workload

Delete the sample deployment and wait for its PodVM to be removed:

```bash
kubectl delete deployment nginx
```

Before uninstalling CAA, find any other pods that use `kata-remote` and delete them. 

For example, the following command finds all pods that use `kata-remote` and deletes them:

```bash
kubectl get pods -A -o custom-columns='NAME:.metadata.name,NAMESPACE:.metadata.namespace,RUNTIMECLASS:.spec.runtimeClassName' \
    | grep kata-remote \
    | awk '{print $1, $2}' \
    | xargs -n 2 sh -c 'kubectl delete pod -n "$2" "$1"' _
```

## Uninstall CAA

List deployed Confidential Containers Helm chart:

> **Note**: This command assumes that only one Helm release is deployed in the `confidential-containers-system` namespace.
> If there are multiple releases, you may need to adjust the command to select the correct one.

```bash
export HELM_COCO_CHART_NAME=$(helm list \
                              -n confidential-containers-system \
                              --short)
```

Delete Confidential Containers related Helm chart:

```bash
helm uninstall ${HELM_COCO_CHART_NAME} \
  --namespace confidential-containers-system
```

Delete Confidential Containers related namespace:

```bash
kubectl delete namespace confidential-containers-system
```

This does not delete the Kubernetes cluster, cloud identities, networks, images, or other provider resources. 
Return to your provider guide to remove those resources.

