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

First, clone the Trustee operator.

```bash
git clone https://github.com/confidential-containers/trustee-operator.git
```

Install the operator.
```bash
make deploy IMG=quay.io/confidential-containers/trustee-operator:latest
```

Verify that the controller is running.
```bash
kubectl get pods -n trustee-operator-system --watch
```

The operator controller should be running.
```bash
NAME                                                   READY   STATUS    RESTARTS   AGE
trustee-operator-controller-manager-6fb5bb5bd9-22wd6   2/2     Running   0          25s
```

### Deploy Trustee

A simple configuration is provided.
You will need to generate an authentication key.

```bash
cd config/samples/microservices
# or config/samples/all-in-one for the integrated mode

# create authentication keys
openssl genpkey -algorithm ed25519 > privateKey
openssl pkey -in privateKey -pubout -out kbs.pem

# create all the needed resources
kubectl apply -k .
```

Check that the Trustee deployment is running.
```bash
kubectl get pods -n trustee-operator-system --selector=app=kbs
```

The Trustee deployment should be running.
```bash
NAME                                  READY   STATUS    RESTARTS   AGE
trustee-deployment-78bd97f6d4-nxsbb   3/3     Running   0          4m3s
```

### Uninstall

Remove the Trustee CRD.
```bash
make uninstall
```

Remove the controller.
```bash
make undeploy
```
