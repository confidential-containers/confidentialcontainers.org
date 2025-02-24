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

The operator (release v0.3.0 at the time of writing) is available in the [Operator Hub](https://operatorhub.io/operator/trustee-operator).

Please follow the installation steps detailed [here](https://confidentialcontainers.org/blog/2024/06/10/deploy-trustee-in-kubernetes/#kubernetes-deployment).

Verify that the controller is running.
```bash
kubectl get pods -n trustee-operator-system --watch
```

The operator controller should be running.
```bash
NAME                                                   READY   STATUS    RESTARTS   AGE
trustee-operator-controller-manager-77cb448dc-7vxck    1/1     Running   0          11m
```

### Deploy Trustee

An example on how to configure trustee is provided in this [blog](https://confidentialcontainers.org/blog/2024/06/10/deploy-trustee-in-kubernetes/#configuration).

After the last configuration step, check that the Trustee deployment is running.
```bash
kubectl get pods -n trustee-operator-system --selector=app=kbs
```

The Trustee deployment should be running.
```bash
NAME                                  READY   STATUS    RESTARTS   AGE
trustee-deployment-f97fb74d6-w5qsm    1/1     Running   0          25m
```

### Uninstall

Remove the Trustee CRD.
```bash
CR_NAME=$(kubectl get kbsconfig -n trustee-operator-system -o=jsonpath='{.items[0].metadata.name}') && kubectl delete KbsConfig $CR_NAME -n trustee-operator-system
```

Remove the controller.
```bash
kubectl delete Subscription -n trustee-operator-system my-trustee-operator
```
