---
title: KBS backed by AKV
description: This documentation describes how to mount secrets stored in Azure Key Vault into a KBS deployment
categories:
- docs
tags:
- docs
- kbs
- azure
- akv
---

## Premise

### AKS

We assume an AKS cluster configured with **Workload Identity** and **Key Vault Secrets Provider**. The former provides a KBS pod with the privileges to access an Azure Key Vault (AKV) instance. The latter is an implementation of Kubernetes' [Secret Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io), mapping secrets from external key vaults into pods. The guides below provide instructions on how to configure a cluster accordingly:

- [Use the Azure Key Vault provider for Secrets Store CSI Driver in an Azure Kubernetes Service (AKS) cluster](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)
- [Use Microsoft Entra Workload ID with Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/learn/tutorial-kubernetes-workload-identity)

### AKV

There should be an AKV instance that has been configured with role based access control (RBAC), containing two secrets named `coco_one` `coco_two` for the purpose of the example. Find out how to configure your instance for RBAC in the guide below.

[Provide access to Key Vault keys, certificates, and secrets with an Azure role-based access control](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide)

> **Note**: You might have to toggle between Access Policy and RBAC modes to create your secrets on the CLI or via the Portal if your user doesn't have the necessary role assignments.

### CoCo

While the steps describe a deployment of KBS, the configuration of a Confidential Containers environment is out of scope for this document. CoCo should be configured with KBS as a Key Broker Client (KBC) and the resulting KBS deployment should be available and configured for confidential pods.

### Azure environment

Configure your Resource group, Subscription and AKS cluster name. Adjust accordingly:

```bash
export SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
export RESOURCE_GROUP=my-group
export KEYVAULT_NAME=kbs-secrets
export CLUSTER_NAME=coco
```

## Instructions

### Create Identity

Create a User managed identity for KBS:

```bash
az identity create --name kbs -g "$RESOURCE_GROUP"
export KBS_CLIENT_ID="$(az identity show -g "$RESOURCE_GROUP" --name kbs --query clientId -o tsv)"
export KBS_TENANT_ID=$(az aks show --name "$CLUSTER_NAME" --resource-group "$RESOURCE_GROUP" --query identity.tenantId -o tsv)
```

Assign a role to access secrets:

```bash
export KEYVAULT_SCOPE=$(az keyvault show --name "$KEYVAULT_NAME" --query id -o tsv)
az role assignment create --role "Key Vault Administrator" --assignee "$KBS_CLIENT_ID" --scope "$KEYVAULT_SCOPE"
```

### Namespace

By default KBS is deployed into a `coco-tenant` Namespace:

```bash
export NAMESPACE=coco-tenant
kubectl create namespace $NAMESPACE
```

### KBS identity and Service Account

Workload Identity provides individual pods with IAM privileges to access Azure infrastructure resources. An azure identity is bridged to a Service Account using OIDC and Federated Credentials. Those are scoped to a Namespace, we assume we deploy the Service Account and KBS into the `default` Namespace, adjust accordingly if necessary.

```bash
export AKS_OIDC_ISSUER="$(az aks show --resource-group "$RESOURCE_GROUP" --name "$CLUSTER_NAME" --query "oidcIssuerProfile.issuerUrl" -o tsv)"
az identity federated-credential create \
 --name kbsfederatedidentity \
 --identity-name kbs \
 --resource-group "$RESOURCE_GROUP" \
 --issuer "$AKS_OIDC_ISSUER" \
 --subject "system:serviceaccount:${NAMESPACE}:kbs"
```

Create a Service Account object and annotate it with the identity's client id.

```bash
cat <<EOF> service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${KBS_CLIENT_ID}
  name: kbs
  namespace: ${NAMESPACE}
EOF
kubectl apply -f service-account.yaml
```

### Secret Provider Class

A Secret Provider Class specifies a set of secrets that should be made available to k8s workloads.

```bash
cat <<EOF> secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: ${KEYVAULT_NAME}
  namespace: ${NAMESPACE}
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: ${KBS_CLIENT_ID}
    keyvaultName: ${KEYVAULT_NAME}
    objects: |
      array:
      - |
        objectName: coco_one
        objectType: secret
      - |
        objectName: coco_two
        objectType: secret
    tenantId: ${KBS_TENANT_ID}
EOF
kubectl create -f secret-provider-class.yaml
```

### Deploy KBS

The default KBS deployment needs to be extended with label annotations and CSI volume. The secrets are mounted into the storage hierarchy `default/akv`.

```bash
git clone https://github.com/confidential-containers/kbs.git
cd kbs
git checkout v0.8.2
cd kbs/config/kubernetes
mkdir akv
cat <<EOF> akv/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: coco-tenant

resources:
- ../base

patches:
- path: patch.yaml
  target:
    group: apps
    kind: Deployment
    name: kbs
    version: v1
EOF
cat <<EOF> akv/patch.yaml
- op: add
  path: /spec/template/metadata/labels/azure.workload.identity~1use
  value: "true"
- op: add
  path: /spec/template/spec/serviceAccountName
  value: kbs
- op: add
  path: /spec/template/spec/containers/0/volumeMounts/-
  value:
    name: secrets
    mountPath: /opt/confidential-containers/kbs/repository/default/akv
    readOnly: true
- op: add
  path: /spec/template/spec/volumes/-
  value:
    name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: ${KEYVAULT_NAME}
EOF
kubectl apply -k akv/
```

## Test

The KBS pod should be running, the pod events should give indication of possible errors. From a confidential pod the AKV secrets should be retrievable via Confidential Data Hub:

```bash
$ kubectl exec -it deploy/nginx-coco -- curl http://127.0.0.1:8006/cdh/resource/default/akv/coco_one
a secret
```
