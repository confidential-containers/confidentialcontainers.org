---
date: 2024-06-10
title: Deploy Trustee in Kubernetes
linkTitle: Deploy Trustee in Kubernetes
description: >
  Introduction to the trustee-operator for deploying Trustee in a Kubernetes cluster.
author: "[Leonardo Milleri](https://github.com/lmilleri) & [Pradipta Banerjee](https://www.linkedin.com/in/bpradipt/)"
---

## Introduction

In this blog, we’ll be going through the deployment of [Trustee](https://github.com/confidential-containers/trustee), the Key Broker Service that provides keys/secrets to clients that want to execute workloads confidentially.
Trustee provides a built-in attestation service that complies to the [RATS](https://www.ietf.org/archive/id/draft-ietf-rats-architecture-22.html) specification.

In this document, we’ll be focusing on how to deploy Trustee in Kubernetes using the [Trustee operator](https://github.com/confidential-containers/trustee-operator).

{{% alert color="info" %}}
It is highly recommended to refer to the [Introducing Confidential Containers Trustee: Attestation Services Solution Overview and Use Cases](https://www.redhat.com/en/blog/introducing-confidential-containers-trustee-attestation-services-solution-overview-and-use-cases) prior to deploying it.
{{% /alert %}}

## Definitions

First of all, let's introduce some definitions.

In confidential computing environments, **Attestation** is crucial in verifying the trustworthiness of the location where you plan to run your workload.

The **Attester** provides **Evidence**, which is evaluated and appraised to decide its trustworthiness.

The **Endorser** is the HW manufacturer who provides an **endorsement**, which the **verifier** uses to validate the evidence received from the attester.

The **reference value provider service** (RVPS) is a component in the **Attestation Service** (AS) responsible for storing and providing reference values.

## Kubernetes deployment

The following instructions are assuming a Kubernetes cluster is set up with the Operator Lifecycle Manager (OLM) running. OLM helps users install, update, and manage the lifecycle of Kubernetes native applications (Operators) and their associated services.

{{% alert color="info" %}}
If we don't have a running cluster yet, we can easily bring it up with [kind](https://kind.sigs.k8s.io/docs/user/quick-start#installation). For example:
{{% /alert %}}

```bash
kind create cluster -n trustee
# install the olm operator
kubectl create -f https://raw.githubusercontent.com/operator-framework/operator-lifecycle-manager/master/deploy/upstream/quickstart/crds.yaml
kubectl create -f https://raw.githubusercontent.com/operator-framework/operator-lifecycle-manager/master/deploy/upstream/quickstart/olm.yaml
```

{{% alert color="info" %}}
For more information on the Operator Lifecycle Manager (OLM) and the operator installation procedure from OperatorHub.io, please consult this [guide](https://operatorhub.io/how-to-install-an-operator).
{{% /alert %}}


### Namespace creation

This is the default Namespace, where all the relevant Trustee objects will be created.

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Namespace
metadata:
  name: kbs-operator-system
EOF
```

### Operator Group

An Operator group, defined by the OperatorGroup resource, provides multi-tenant configuration to OLM-installed Operators:

```bash
kubectl apply -f - << EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kbs-operator-system
  namespace: kbs-operator-system
spec:
EOF
```

### Subscription

A subscription, defined by a Subscription object, represents an intention to install an Operator. It is the custom resource that relates an Operator to a catalog source:

```bash
kubectl apply -f - << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kbs-operator-system
  namespace: kbs-operator-system
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: trustee-operator
  source: operatorhubio-catalog
  sourceNamespace: olm
  startingCSV: trustee-operator.v0.1.0
EOF
```

{{% alert color="info" %}}
Note that the Trustee operator has been already published in the operator [hub catalog](https://operatorhub.io/).
{{% /alert %}}

### Check Trustee Operator installation

Now it is time to check if the Trustee operator has been installed properly, by running the command:

```bash
kubectl get csv -n kbs-operator-system
```

We should expect something like:

```bash
NAME                      DISPLAY            VERSION   REPLACES   PHASE
trustee-operator.v0.1.0   Trustee Operator   0.1.0                Succeeded
```

## Configuration

The Trustee Operator configuration requires a few steps. Some of the steps are provided as an example, but you may want to customize the examples for your real requirements.

### Authorization key-pair generation

First of all, we’d need to create the key pairs for Trustee authorization. The public key is used by Trustee for client authorization, the private key is used by the client to prove its identity and register keys/secrets.


Create secret for client authorization:

```bash
openssl genpkey -algorithm ed25519 > privateKey
openssl pkey -in privateKey -pubout -out publicKey
kubectl create secret generic kbs-auth-public-key --from-file=publicKey -n kbs-operator-system
```

### HTTPS configuration

It is recommended to enable the HTTPS protocol for the following reasons:
- secure the Trustee server API
- bind the Trusted Execution Environment (TEE) to a given Trustee server by seeding the public key and certificate (as measured init data)
 
In this example we're going to create a self-signed certificate using the following template:

```bash
cat << EOF > kbs-service-509.conf
[req]
default_bits       = 2048
default_keyfile    = localhost.key
distinguished_name = req_distinguished_name
req_extensions     = req_ext
x509_extensions    = v3_ca

[req_distinguished_name]
countryName                 = Country Name (2 letter code)
countryName_default         = UK
stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = England
localityName                = Locality Name (eg, city)
localityName_default        = Bristol
organizationName            = Organization Name (eg, company)
organizationName_default    = Red Hat
organizationalUnitName      = organizationalunit
organizationalUnitName_default = Development
commonName                  = Common Name (e.g. server FQDN or YOUR name)
commonName_default          = kbs-service
commonName_max              = 64

[req_ext]
subjectAltName = @alt_names

[v3_ca]
subjectAltName = @alt_names

[alt_names]
DNS.1   = kbs-service
EOF
```

Create secret for self-signed certificate:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout https.key -out https.crt \
    -config kbs-service-509.conf -passin pass:\
    -subj "/C=UK/ST=England/L=Bristol/O=Red Hat/OU=Development/CN=kbs-service"
kubectl create secret generic kbs-https-certificate --from-file=https.crt -n kbs-operator-system
kubectl create secret generic kbs-https-key --from-file=https.key -n kbs-operator-system
```

### Trustee ConfigMap object

This command will create the ConfigMap object that provides Trustee all the needed configuration:

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: kbs-config
  namespace: kbs-operator-system
data:
  kbs-config.json: |
    {
        "insecure_http" : false,
        "private_key": "/etc/https-key/https.key",
        "certificate": "/etc/https-cert/https.crt",
        "sockets": ["0.0.0.0:8080"],
        "auth_public_key": "/etc/auth-secret/publicKey",
        "attestation_token_config": {
          "attestation_token_type": "CoCo"
        },
        "repository_config": {
          "type": "LocalFs",
          "dir_path": "/opt/confidential-containers/kbs/repository"
        },
        "as_config": {
          "work_dir": "/opt/confidential-containers/attestation-service",
          "policy_engine": "opa",
          "attestation_token_broker": "Simple",
          "attestation_token_config": {
            "duration_min": 5
          },
          "rvps_config": {
            "store_type": "LocalJson",
            "store_config": {
              "file_path": "/opt/confidential-containers/rvps/reference-values/reference-values.json"
            }
          }
        },
        "policy_engine_config": {
          "policy_path": "/opt/confidential-containers/opa/policy.rego"
        }
    }
EOF
```

### Reference Values

The reference values are an important part of the attestation process. The client collects the measurements (from the running software, the TEE hardware and its firmware) and submits a quote with the claims to the attestation server. These measurements, in order for the attestation protocol to succeed, have to match one of potentially multiple configured valid values that had been registered to Trustee previously. You could also apply flexible rules like "firmware of secure processor > v1.30", etc. This process guarantees the cVM (confidential VM) is running the expected software stack and that it hasn’t been tampered with.

{{% alert color="info" %}}
The following command shows how to registers reference values in RVPS. Note that in this case an empty list is provided, because the sample attester doesn't handle real measurements. According to your HW platform, you'd need to register some real trusted digests instead.
{{% /alert %}}


```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: rvps-reference-values
  namespace: kbs-operator-system
data:
  reference-values.json: |
    [
    ]
EOF
```

### Create secrets

How to create secrets to be shared with the attested clients?
In this example we create a secret *kbsres1* with two entries. These resources (key1, key2) can be retrieved by the Trustee clients.
You can add more secrets as per your requirements.

```bash
kubectl create secret generic kbsres1 --from-literal key1=res1val1 --from-literal key2=res1val2 -n kbs-operator-system
```

### Create KbsConfig CRD

Finally, the CRD for the operator is created:

```bash
kubectl apply -f - << EOF
apiVersion: confidentialcontainers.org/v1alpha1
kind: KbsConfig
metadata:
  labels:
    app.kubernetes.io/name: kbsconfig
    app.kubernetes.io/instance: kbsconfig-sample
    app.kubernetes.io/part-of: kbs-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: kbs-operator
  name: kbsconfig-sample
  namespace: kbs-operator-system
spec:
  kbsConfigMapName: kbs-config
  kbsAuthSecretName: kbs-auth-public-key
  kbsDeploymentType: AllInOneDeployment
  kbsRvpsRefValuesConfigMapName: rvps-reference-values
  kbsSecretResources: ["kbsres1"]
  kbsHttpsKeySecretName: kbs-https-key
  kbsHttpsCertSecretName: kbs-https-certificate
EOF
```

### Set Namespace for the context entry

```bash
kubectl config set-context --current --namespace=kbs-operator-system
```

### Check if the PODs are running

```bash
kubectl get pods -n kbs-operator-system
NAME                                                   READY   STATUS    RESTARTS   AGE
trustee-deployment-7bdc6858d7-bdncx                    1/1     Running   0          69s
trustee-operator-controller-manager-6c584fc969-8dz2d   2/2     Running   0          4h7m
```

Also, the log should report something like:

```bash
POD_NAME=$(kubectl get pods -l app=kbs -o jsonpath='{.items[0].metadata.name}' -n kbs-operator-system)
kubectl logs -n kbs-operator-system $POD_NAME
[2024-06-10T13:38:01Z INFO  kbs] Using config file /etc/kbs-config/kbs-config.json
[2024-06-10T13:38:01Z WARN  attestation_service::rvps] No RVPS address provided and will launch a built-in rvps
[2024-06-10T13:38:01Z INFO  attestation_service::token::simple] No Token Signer key in config file, create an ephemeral key and without CA pubkey cert
[2024-06-10T13:38:01Z INFO  api_server] Starting HTTPS server at [0.0.0.0:8080]
[2024-06-10T13:38:01Z INFO  actix_server::builder] starting 12 workers
[2024-06-10T13:38:01Z INFO  actix_server::server] Tokio runtime found; starting in existing Tokio runtime
```

## End-to-End Attestation

Since we're running this tutorial in a regular machine (no HW endorsement), we need to customize the default resource policy when using the sample attester (no real HW TEE platform).
In the default policy, claims originating from a `sample` TEE would be rejected. This restriction should not be removed in a production scenario.

To showcase how we can assert properties of a TEE, we assert the sample TEE's "security version number". For a real TEE this could be a minimum firmware revision, or similar properties of a TEE.

```bash
cat << EOF > policy.rego
package policy

default allow = false
allow {
        input["tcb-status"]["sample.svn"] == "1"
}
EOF

POD_NAME=$(kubectl get pods -l app=kbs -o jsonpath='{.items[0].metadata.name}' -n kbs-operator-system)
kubectl cp --no-preserve policy.rego $POD_NAME:/opt/confidential-containers/opa/policy.rego
```

We create a pod using an already existing image where the kbs-client is deployed:

```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: kbs-client
spec:
  containers:
  - name: kbs-client
    image: quay.io/confidential-containers/kbs-client:latest
    imagePullPolicy: IfNotPresent
    command:
      - sleep
      - "360000"
    env:
      - name: RUST_LOG
        value:  none
EOF
```

Finally we are able to test the entire attestation protocol, when fetching one of the aforementioned secret:

```bash
kubectl cp https.crt kbs-client:/
kubectl exec -it kbs-client -- kbs-client --cert-file https.crt --url https://kbs-service:8080 get-resource --path default/kbsres1/key1
cmVzMXZhbDE=
```

If we type the command:

```bash
echo cmVzMXZhbDE= | base64 -d
```

We’ll get *res1val1*, the secret we created before.

## Summary

In this blog we have shown how to use the Trustee operator for deploying Trustee and run the attestation workflow with a sample attester.