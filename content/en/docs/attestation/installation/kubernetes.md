---
title: Trustee Operator
description: Installing Trustee on Kubernetes
weight: 20
categories:
- attestation
tags:
- trustee
- attestation
- installation
- kubernetes
---

Trustee can be installed on Kubernetes using the Trustee operator.
When running Trustee in Kubernetes with the operator, the cluster must be Trusted.

### Install the operator

The operator (release v0.17.0 at the time of writing) is available in the [Operator Hub](https://operatorhub.io/operator/trustee-operator).

Please follow the installation steps detailed [here](https://confidentialcontainers.org/blog/2026/02/11/deploy-trustee-in-kubernetes/#kubernetes-deployment).

Verify that the controller is running.
```bash
kubectl get pods -n operators --watch
```

The operator controller should be running.
```bash
NAME                                                   READY   STATUS    RESTARTS   AGE
trustee-operator-controller-manager-77cb448dc-7vxck    1/1     Running   0          11m
```

#### How to override the Trustee image

First of all we need to know which Trustee image is running:

```bash
kubectl get csv -n operators trustee-operator.v0.17.0 -o json | jq '.spec.install.spec.deployments[0].spec.template.spec.containers[0].env[1].value'
"ghcr.io/confidential-containers/key-broker-service:built-in-as-v0.16.0"
```

The default image can be replaced with an updated version, for example Trustee v0.17.0:

```bash
NEW_IMAGE=ghcr.io/confidential-containers/key-broker-service:built-in-as-v0.17.0
kubectl patch csv -n operators trustee-operator.v0.17.0 --type='json' -p="[{'op': 'replace', 'path': '/spec/install/spec/deployments/0/spec/template/spec/containers/0/env/1/value', 'value':$NEW_IMAGE}]"
```

### Deploy Trustee

An example on how to configure Trustee is provided in this [blog](https://confidentialcontainers.org/blog/2026/02/11/deploy-trustee-in-kubernetes/#configuration).

After the last configuration step, check that the Trustee deployment is running.
```bash
kubectl get pods -n operators --selector=app=kbs
```

The Trustee deployment should be running.
```bash
NAME                                  READY   STATUS    RESTARTS   AGE
trustee-deployment-f97fb74d6-w5qsm    1/1     Running   0          25m
```

### Uninstall

Remove the Trustee CRD.
```bash
CR_NAME=$(kubectl get kbsconfig -n operators -o=jsonpath='{.items[0].metadata.name}') && kubectl delete KbsConfig $CR_NAME -n operators
```

Remove the controller.
```bash
kubectl delete Subscription -n operators my-trustee-operator
kubectl delete csv -n operators trustee-operator.v0.3.0
```
