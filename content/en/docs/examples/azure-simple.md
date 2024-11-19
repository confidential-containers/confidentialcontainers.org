---
title: Azure
description: Cloud API Adaptor (CAA) on Azure
categories:
- examples
tags:
- caa
- azure
---

This documentation will walk you through setting up CAA (a.k.a. Peer Pods) on Azure Kubernetes Service (AKS). It explains how to deploy:

- A single worker node Kubernetes cluster using Azure Kubernetes Service (AKS)
- CAA on that Kubernetes cluster
- An Nginx pod backed by CAA pod VM

## Pre-requisites

- Install Azure CLI by following instructions [here](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).
- Install kubectl by following the instructions [here](https://kubernetes.io/docs/tasks/tools/#kubectl).
- Ensure that the tools `curl`, `git`, `jq` and `sipcalc` are installed.

## Azure Preparation

### Azure login

There are a bunch of steps that require you to be logged into your Azure account:

```bash
az login
```

Retrieve your subscription ID:

```bash
export AZURE_SUBSCRIPTION_ID=$(az account show --query id --output tsv)
```

Set the region:

{{< tabpane text=true right=true persist=header >}}

{{% tab header="AMD SEV-SNP" %}}

```bash
export AZURE_REGION="eastus"
```

> **Note:** We selected the `eastus` region as it not only offers AMD SEV-SNP machines but also has prebuilt pod VM images readily available.

{{% /tab %}}

{{% tab header="Intel TDX" %}}

```bash
export AZURE_REGION="eastus2"
```

> **Note:** We selected the `eastus2` region as it not only offers Intel TDX machines but also has prebuilt pod VM images readily available.

{{% /tab %}}

{{% tab header="Non-Confidential" %}}

```bash
export AZURE_REGION="eastus"
```

> **Note:** We have chose region `eastus` because it has prebuilt pod VM images readily available.

{{% /tab %}}
{{< /tabpane >}}

### Resource group

> **Note**: Skip this step if you already have a resource group you want to use. Please, export the resource group name in the `AZURE_RESOURCE_GROUP` environment variable.

Create an Azure resource group by running the following command:

```bash
export AZURE_RESOURCE_GROUP="caa-rg-$(date '+%Y%m%b%d%H%M%S')"

az group create \
  --name "${AZURE_RESOURCE_GROUP}" \
  --location "${AZURE_REGION}"
```

### Deploy Kubernetes using AKS

Make changes to the following environment variable as you see fit:

```bash
export CLUSTER_NAME="caa-$(date '+%Y%m%b%d%H%M%S')"
export AKS_WORKER_USER_NAME="azuser"
export AKS_RG="${AZURE_RESOURCE_GROUP}-aks"
export SSH_KEY=~/.ssh/id_rsa.pub
```

> **Note**: Optionally, deploy the worker nodes into an existing Azure Virtual Network (VNet) and subnet by adding the following flag: `--vnet-subnet-id $MY_SUBNET_ID`.

Deploy AKS with single worker node to the same resource group you created earlier:

```bash
az aks create \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --node-resource-group "${AKS_RG}" \
  --name "${CLUSTER_NAME}" \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --location "${AZURE_REGION}" \
  --node-count 1 \
  --node-vm-size Standard_F4s_v2 \
  --nodepool-labels node.kubernetes.io/worker= \
  --ssh-key-value "${SSH_KEY}" \
  --admin-username "${AKS_WORKER_USER_NAME}" \
  --os-sku Ubuntu
```

Download kubeconfig locally to access the cluster using `kubectl`:

```bash
az aks get-credentials \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --name "${CLUSTER_NAME}"
```

### User assigned identity and federated credentials

CAA needs privileges to talk to Azure API. This privilege is granted to CAA by associating a workload identity to the CAA service account. This workload identity (a.k.a. user assigned identity) is given permissions to create VMs, fetch images and join networks in the next step.

> **Note**: If you use an existing AKS cluster it might need to be configured to support workload identity and OpenID Connect (OIDC), please refer to the instructions in [this guide](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster#update-an-existing-aks-cluster).

Start by creating an identity for CAA:

```bash
export AZURE_WORKLOAD_IDENTITY_NAME="caa-${CLUSTER_NAME}"

az identity create \
  --name "${AZURE_WORKLOAD_IDENTITY_NAME}" \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --location "${AZURE_REGION}"
```

```bash
export USER_ASSIGNED_CLIENT_ID="$(az identity show \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --name "${AZURE_WORKLOAD_IDENTITY_NAME}" \
  --query 'clientId' \
  -otsv)"
```

### Networking

The VMs that will host Pods will commonly require access to internet services, e.g. to pull images from a public OCI registry. A discrete subnet can be created next to the AKS cluster subnet in the same VNet. We then attach a NAT gateway with a public IP to that subnet:


```bash
export AZURE_VNET_NAME="$(az network vnet list -g ${AKS_RG} --query '[].name' -o tsv)"
export AKS_CIDR="$(az network vnet show -n $AZURE_VNET_NAME -g $AKS_RG --query "subnets[?name == 'aks-subnet'].addressPrefix" -o tsv)"
# 10.224.0.0/16
export MASK="${AKS_CIDR#*/}"
# 16
PEERPOD_CIDR="$(sipcalc $AKS_CIDR -n 2 | grep ^Network | grep -v current | cut -d' ' -f2)/${MASK}"
# 10.225.0.0/16
az network public-ip create -g "$AKS_RG" -n peerpod
az network nat gateway create -g "$AKS_RG" -l "$AZURE_REGION" --public-ip-addresses peerpod -n peerpod
az network vnet subnet create -g "$AKS_RG" --vnet-name "$AZURE_VNET_NAME" --nat-gateway peerpod --address-prefixes "$PEERPOD_CIDR" -n peerpod
export AZURE_SUBNET_ID="$(az network vnet subnet show -g "$AKS_RG" --vnet-name "$AZURE_VNET_NAME" -n peerpod --query id -o tsv)"
```

### AKS resource group permissions

For CAA to be able to manage VMs assign the identity VM and Network contributor roles, privileges to spawn VMs in `$AZURE_RESOURCE_GROUP` and attach to a VNet in `$AKS_RG`.

```bash
az role assignment create \
  --role "Virtual Machine Contributor" \
  --assignee "$USER_ASSIGNED_CLIENT_ID" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}/resourcegroups/${AZURE_RESOURCE_GROUP}"
```

```bash
az role assignment create \
  --role "Reader" \
  --assignee "$USER_ASSIGNED_CLIENT_ID" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}/resourcegroups/${AZURE_RESOURCE_GROUP}"
```

```bash
az role assignment create \
  --role "Network Contributor" \
  --assignee "$USER_ASSIGNED_CLIENT_ID" \
  --scope "/subscriptions/${AZURE_SUBSCRIPTION_ID}/resourcegroups/${AKS_RG}"
```

Create the federated credential for the CAA ServiceAccount using the OIDC endpoint from the AKS cluster:

```bash
export AKS_OIDC_ISSUER="$(az aks show \
  --name "${CLUSTER_NAME}" \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --query "oidcIssuerProfile.issuerUrl" \
  -otsv)"
```

```bash
az identity federated-credential create \
  --name "caa-${CLUSTER_NAME}" \
  --identity-name "${AZURE_WORKLOAD_IDENTITY_NAME}" \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --issuer "${AKS_OIDC_ISSUER}" \
  --subject system:serviceaccount:confidential-containers-system:cloud-api-adaptor \
  --audience api://AzureADTokenExchange
```

## Deploy CAA

> **Note**: If you are using Calico Container Network Interface (CNI) on the Kubernetes cluster, then, [configure](https://projectcalico.docs.tigera.io/networking/vxlan-ipip#configure-vxlan-encapsulation-for-all-inter-workload-traffic) Virtual Extensible LAN (VXLAN) encapsulation for all inter workload traffic.

### Download the CAA deployment artifacts

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

```bash
export CAA_VERSION="0.10.0"
curl -LO "https://github.com/confidential-containers/cloud-api-adaptor/archive/refs/tags/v${CAA_VERSION}.tar.gz"
tar -xvzf "v${CAA_VERSION}.tar.gz"
cd "cloud-api-adaptor-${CAA_VERSION}/src/cloud-api-adaptor"
```

{{% /tab %}}

{{% tab header="Latest Build" %}}

```bash
export CAA_BRANCH="main"
curl -LO "https://github.com/confidential-containers/cloud-api-adaptor/archive/refs/heads/${CAA_BRANCH}.tar.gz"
tar -xvzf "${CAA_BRANCH}.tar.gz"
cd "cloud-api-adaptor-${CAA_BRANCH}/src/cloud-api-adaptor"
```

{{% /tab %}}

{{% tab header="DIY" %}}
This assumes that you already have the code ready to use. On your terminal change directory to the Cloud API Adaptor's code base.
{{% /tab %}}

{{< /tabpane >}}

### CAA pod VM image

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

Export this environment variable to use for the peer pod VM:

```bash
export AZURE_IMAGE_ID="/CommunityGalleries/cococommunity-42d8482d-92cd-415b-b332-7648bd978eff/Images/peerpod-podvm-ubuntu2204-cvm-snp/Versions/${CAA_VERSION}"
```

{{% /tab %}}

{{% tab header="Latest Build" %}}

An automated job builds the pod VM image each night at 00:00 UTC. You can use that image by exporting the following environment variable:

```bash
SUCCESS_TIME=$(curl -s \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/confidential-containers/cloud-api-adaptor/actions/workflows/azure-podvm-image-nightly-build.yml/runs?status=success" \
  | jq -r '.workflow_runs[0].updated_at')

export AZURE_IMAGE_ID="/CommunityGalleries/cocopodvm-d0e4f35f-5530-4b9c-8596-112487cdea85/Images/podvm_image0/Versions/$(date -u -jf "%Y-%m-%dT%H:%M:%SZ" "$SUCCESS_TIME" "+%Y.%m.%d" 2>/dev/null || date -d "$SUCCESS_TIME" +%Y.%m.%d)"
```

Above image version is in the format `YYYY.MM.DD`, so to use the latest image should be today's date or yesterday's date.

{{% /tab %}}

{{% tab header="DIY" %}}

If you have made changes to the CAA code that affects the pod VM image and you want to deploy those changes then follow [these instructions](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/azure/build-image.md) to build the pod VM image. Once image build is finished then export image id to the environment variable `AZURE_IMAGE_ID`.

{{% /tab %}}

{{< /tabpane >}}

### CAA container image

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

Export the following environment variable to use the latest release image of CAA:

```bash
export CAA_IMAGE="quay.io/confidential-containers/cloud-api-adaptor"
export CAA_TAG="v0.10.0-amd64"
```

{{% /tab %}}

{{% tab header="Latest Build" %}}

Export the following environment variable to use the image built by the CAA CI on each merge to main:

```bash
export CAA_IMAGE="quay.io/confidential-containers/cloud-api-adaptor"
```

Find an appropriate tag of pre-built image suitable to your needs [here](https://quay.io/repository/confidential-containers/cloud-api-adaptor?tab=tags&tag=latest).

```bash
export CAA_TAG=""
```

> **Caution**: You can also use the `latest` tag but it is **not** recommended, because of its lack of version control and potential for unpredictable updates, impacting stability and reproducibility in deployments.

{{% /tab %}}

{{% tab header="DIY" %}}

If you have made changes to the CAA code and you want to deploy those changes then follow [these instructions](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/README.md#building-custom-cloud-api-adaptor-image) to build the container image. Once the image is built export the environment variables `CAA_IMAGE` and `CAA_TAG`.

{{% /tab %}}

{{< /tabpane >}}

### Annotate Service Account

Annotate the CAA Service Account with the workload identity's `CLIENT_ID` and make the CAA DaemonSet use workload identity for authentication:

```yaml
cat <<EOF > install/overlays/azure/workload-identity.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cloud-api-adaptor-daemonset
  namespace: confidential-containers-system
spec:
  template:
    metadata:
      labels:
        azure.workload.identity/use: "true"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloud-api-adaptor
  namespace: confidential-containers-system
  annotations:
    azure.workload.identity/client-id: "$USER_ASSIGNED_CLIENT_ID"
EOF
```

### Select peer-pods machine type

{{< tabpane text=true right=true persist=header >}}
{{% tab header="AMD SEV-SNP" %}}

```bash
export AZURE_INSTANCE_SIZE="Standard_DC2as_v5"
export DISABLECVM="false"
```

Find more AMD SEV-SNP machine types on [this](https://learn.microsoft.com/en-us/azure/virtual-machines/dasv5-dadsv5-series) Azure documentation.

{{% /tab %}}

{{% tab header="Intel TDX" %}}

```bash
export AZURE_INSTANCE_SIZE="Standard_DC2es_v5"
export DISABLECVM="false"
```

Find more Intel TDX machine types on [this](https://learn.microsoft.com/en-us/azure/virtual-machines/dcesv5-dcedsv5-series) Azure documentation.

{{% /tab %}}

{{% tab header="Non-Confidential" %}}

```bash
export AZURE_INSTANCE_SIZE="Standard_D2as_v5"
export DISABLECVM="true"
```

{{% /tab %}}
{{< /tabpane >}}

### Populate the `kustomization.yaml` file

Run the following command to update the [`kustomization.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/install/overlays/azure/kustomization.yaml) file:

```yaml
cat <<EOF > install/overlays/azure/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../yamls
images:
- name: cloud-api-adaptor
  newName: "${CAA_IMAGE}"
  newTag: "${CAA_TAG}"
generatorOptions:
  disableNameSuffixHash: true
configMapGenerator:
- name: peer-pods-cm
  namespace: confidential-containers-system
  literals:
  - CLOUD_PROVIDER="azure"
  - AZURE_SUBSCRIPTION_ID="${AZURE_SUBSCRIPTION_ID}"
  - AZURE_REGION="${AZURE_REGION}"
  - AZURE_INSTANCE_SIZE="${AZURE_INSTANCE_SIZE}"
  - AZURE_RESOURCE_GROUP="${AZURE_RESOURCE_GROUP}"
  - AZURE_SUBNET_ID="${AZURE_SUBNET_ID}"
  - AZURE_IMAGE_ID="${AZURE_IMAGE_ID}"
  - DISABLECVM="${DISABLECVM}"
secretGenerator:
- name: peer-pods-secret
  namespace: confidential-containers-system
- name: ssh-key-secret
  namespace: confidential-containers-system
  files:
  - id_rsa.pub
patchesStrategicMerge:
- workload-identity.yaml
EOF
```

The SSH public key should be accessible to the `kustomization.yaml` file:

```bash
cp $SSH_KEY install/overlays/azure/id_rsa.pub
```

### Deploy CAA on the Kubernetes cluster

Deploy coco operator:

```bash
export COCO_OPERATOR_VERSION="0.10.0"
kubectl apply -k "github.com/confidential-containers/operator/config/release?ref=v${COCO_OPERATOR_VERSION}"
kubectl apply -k "github.com/confidential-containers/operator/config/samples/ccruntime/peer-pods?ref=v${COCO_OPERATOR_VERSION}"
```

Run the following command to deploy CAA:

```bash
kubectl apply -k "install/overlays/azure"
```

Generic CAA deployment instructions are also described [here](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/install/README.md).

## Run sample application

### Ensure runtimeclass is present

Verify that the `runtimeclass` is created after deploying CAA:

```bash
kubectl get runtimeclass
```

Once you can find a `runtimeclass` named `kata-remote` then you can be sure that the deployment was successful. A successful deployment will look like this:

```console
$ kubectl get runtimeclass
NAME          HANDLER       AGE
kata-remote   kata-remote   7m18s
```

### Deploy workload

Create an `nginx` deployment:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
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
        imagePullPolicy: Always
EOF
```

Ensure that the pod is up and running:

```bash
kubectl get pods -n default
```

You can verify that the peer pod VM was created by running the following command:

```bash
az vm list \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --output table
```

Here you should see the VM associated with the pod `nginx`.

> **Note**: If you run into problems then check the troubleshooting guide [here](../troubleshooting/).

## Cleanup

If you wish to clean up the whole set up, you can delete the resource group by running the following command:

```bash
az group delete \
  --name "${AZURE_RESOURCE_GROUP}" \
  --yes --no-wait
```
