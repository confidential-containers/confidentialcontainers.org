---
title: Azure
description: Peer Pods Helm Chart using Cloud API Adaptor (CAA) on Azure
categories:
  - examples
tags:
  - helm
  - caa
  - azure
  - aks
---

This documentation will walk you through setting up Cloud API Adaptor (CAA) (a.k.a. Peer Pods) on 
Azure Kubernetes Service (AKS). 

It explains how to deploy:

- A single worker node Kubernetes cluster using Azure Kubernetes Service (AKS),
- CAA on that Kubernetes cluster,
- A sample application deployed using CAA to verify that everything is working as expected.

Confidential Containers also supports using Azure Key Vault as a resource backend for Trustee.
[More info](../../attestation/resources/kbs-backed-by-akv)

## Pre-requisites

Install Required Tools:

- Install [kubectl](https://kubernetes.io/docs/tasks/tools/),
- Install [Helm](https://helm.sh/docs/intro/install),
- Install `az` CLI [tool](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli),
- Ensure that the tools `curl`, `git`, `jq` and `sipcalc` are installed.

## Azure Preparation

### Azure login

1. Authenticate with Azure by running the following command and following the instructions in the command line:

   ```bash
   az login
   ```

   > **Note**: If you have access to multiple Azure accounts, make sure to select the one you want to use for this deployment in the command line after running `az login`.

2. Export the Azure subscription ID into `AZ_SUBSCR_ID` using the subscription currently selected in the Azure CLI context:

   ```bash
   export AZ_SUBSCR_ID=$(az account show --query id --output tsv)
   ```

   <details>
      <summary><strong>Verify that the correct subscription is selected</strong></summary>
   
   ```bash
   az account show --query '{name:name, id:id, tenantId:tenantId}' -o yaml
   ```
   </details>

3. Set the `AZURE_REGION` environment variable to the region you want to use for this deployment.
This region will be used to deploy both the AKS cluster and the peer pod VMs.

   {{< tabpane text=true right=true persist=header >}}

{{% tab header="AMD SEV-SNP" %}}

```bash
export AZURE_REGION="eastus"
```

> **Note:** We selected the `eastus` region as it not only offers AMD SEV-SNP machines but also has prebuilt pod VM images readily available.

{{% /tab %}}

{{% tab header="Intel® TDX" %}}

```bash
export AZURE_REGION="eastus"
```

> **Note:** We selected the `eastus` region as it not only offers Intel® TDX machines but also has prebuilt pod VM images readily available.

> **Note:** Currently Intel® TDX machines are available in a limited number of regions:<br>
> (`westeurope`, `westus`, `WestUS3`, `eastus`, `EastUS2EUAP`, `northcentralus`), so make sure to select a region that supports them.

{{% /tab %}}

{{% tab header="Non-Confidential" %}}

```bash
export AZURE_REGION="eastus"
```

> **Note:** We have chose region `eastus` because it has prebuilt pod VM images readily available.

{{% /tab %}}
{{< /tabpane >}}

### Resource group

> **Note**: Skip this step if you already have a resource group you want to use. 
> Please, export the resource group name in the `AZURE_RESOURCE_GROUP` environment variable.

Create an Azure resource group by running the following command:

1. Export the `AZURE_RESOURCE_GROUP` environment variable to a unique name for the resource group:

   ```bash
   export AZURE_RESOURCE_GROUP="caa-rg-$(date '+%Y%m%b%d%H%M%S')"
   ```

2. Create the resource group in the specified region:

   ```bash
   az group create \
     --name "${AZURE_RESOURCE_GROUP}" \
     --location "${AZURE_REGION}"
   ```

### Deploy Kubernetes using AKS

1. Export environment variables related to the AKS cluster deployment:

   ```bash
   export CLUSTER_NAME="caa-$(date '+%Y%m%b%d%H%M%S')"
   export AKS_WORKER_USER_NAME="azuser"
   export AKS_RG="${AZURE_RESOURCE_GROUP}-aks"
   
   # SSH key is optional, but it allows user to SSH into the pod VMs for troubleshooting purposes. 
   # This option works only for custom debug enabled pod VM images. 
   # The prebuilt pod VM images do not have SSH connection enabled.
   export SSH_KEY=~/.ssh/id_rsa.pub
   ```

2. Deploy AKS with single worker node to the same resource group you have created:

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
     --ssh-access disabled \
     --admin-username "${AKS_WORKER_USER_NAME}" \
     --os-sku Ubuntu
   ```

   > **Note**: Optionally, deploy the worker nodes into an existing Azure Virtual Network (VNet) and subnet by adding the following flag: `--vnet-subnet-id <MY_SUBNET_ID>`.
   
   > **Note**: For sample usage `Standard_F4s_v2` machine type is used for the worker node - this machine is General purpose - without enabled confidentiality.

3. Download kubeconfig locally to access the cluster using `kubectl`:

   ```bash
   az aks get-credentials \
     --resource-group "${AZURE_RESOURCE_GROUP}" \
     --name "${CLUSTER_NAME}"
   ```

### User assigned identity and federated credentials

CAA needs privileges to talk to Azure API. 
This privilege is granted to CAA by associating a workload identity to the CAA service account. 
This workload identity (a.k.a. user assigned identity) is given permissions to create VMs, fetch images and join networks in the next step.

> **Note**: If you use an existing AKS cluster it might need to be configured to support workload identity and OpenID Connect (OIDC), please refer to the instructions in [this guide](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster#update-an-existing-aks-cluster).

Create an identity for CAA:

1. Set the `AZURE_WORKLOAD_IDENTITY_NAME` environment variable to a name for the user assigned identity:

    ```bash
    export AZURE_WORKLOAD_IDENTITY_NAME="${CLUSTER_NAME}-identity"
    ```

2. Create the user assigned identity in Azure:

   ```bash
   az identity create \
     --name "${AZURE_WORKLOAD_IDENTITY_NAME}" \
     --resource-group "${AZURE_RESOURCE_GROUP}" \
     --location "${AZURE_REGION}"
   ```

3. Export the client ID of the user assigned identity to the environment variable `USER_ASSIGNED_CLIENT_ID`:

   ```bash
   export USER_ASSIGNED_CLIENT_ID="$(az identity show \
     --resource-group "${AZURE_RESOURCE_GROUP}" \
     --name "${AZURE_WORKLOAD_IDENTITY_NAME}" \
     --query 'clientId' \
     -o tsv)"
   ```

### Networking

The VMs that will host Pods will commonly require access to internet services, e.g. to pull images from a public OCI registry.
Create a discrete subnet next to the AKS cluster subnet in the same VNet for peer pod VMs.
Then create a NAT gateway, attach a public IP, and associate the NAT gateway with the peer-pods subnet to provide outbound connectivity.

1. Get the VNet name of the AKS cluster:

   ```bash
   export AZURE_VNET_NAME="$(az network vnet list \
                               -g ${AKS_RG} \
                               --query '[].name' \
                               -o tsv)"
   ```

2. Define the AKS subnet name.

   By default, the instructions assume `aks-subnet`, but this can differ depending on how the cluster was created.

   ```bash
   export AKS_SUBNET_NAME="aks-subnet"
   ```

   <details>
      <summary><strong>Show how to find subnet names in the VNet</strong></summary>

   ```bash
   az network vnet subnet list \
     -g "${AKS_RG}" \
     --vnet-name "${AZURE_VNET_NAME}" \
     --query '[].name' \
     -o tsv
   ```
   </details>

3. Get the CIDR of the AKS cluster subnet:

   ```bash
   export AKS_CIDR="$(az network vnet show \
     -n $AZURE_VNET_NAME \
     -g $AKS_RG \
     --query "subnets[?name == '${AKS_SUBNET_NAME}'].addressPrefix" \
     -o tsv)"
   ```

   Expected output example: `10.224.0.0/16`.

4. Calculate the CIDR mask for the peer pod subnet by extracting the mask from the AKS CIDR:

   ```bash
   export MASK="${AKS_CIDR#*/}"
   ```

   Expected output example: `16`.

5. Create a CIDR for the peer pod subnet by using `sipcalc` to calculate a new subnet from the AKS CIDR:

   ```bash
   PEERPOD_CIDR="$(
     sipcalc $AKS_CIDR -n 2 \
     | grep ^Network \
     | grep -v current \
     | cut -d' ' -f2
   )/${MASK}"
   ```

   Expected output example: `10.225.0.0/16`.

6. Create a public IP for the NAT gateway:

   ```bash
   az network public-ip create -g "$AKS_RG" -n peerpod
   ```

7. Create a NAT gateway and attach the public IP to it:

   ```bash
   az network nat gateway create \
     -g "$AKS_RG" \
     -l "$AZURE_REGION" \
     --public-ip-addresses peerpod \
     -n peerpod
   ```

8. Create a subnet for the peer pods and attach the NAT gateway to it:

   ```bash
   az network vnet subnet create -g "$AKS_RG" \
     --vnet-name "$AZURE_VNET_NAME" \
     --nat-gateway peerpod \
     --address-prefixes "$PEERPOD_CIDR" \
     -n peerpod
   ```

9. Export the subnet ID to the environment variable `AZURE_SUBNET_ID`:

   ```bash
   export AZURE_SUBNET_ID="$(
     az network vnet subnet show \
     -g "$AKS_RG" \
     --vnet-name "$AZURE_VNET_NAME" \
     -n peerpod \
     --query id \
     -o tsv)"
   ```

### AKS resource group permissions

For CAA to be able to manage VMs, assign Virtual Machine and Network related roles to the user assigned identity.
The roles grant permissions to create VMs in `AZURE_RESOURCE_GROUP` and attach them to the VNet/subnet in `AKS_RG`.

1. Assign the *"Virtual Machine Contributor"* role to the user assigned identity:

   ```bash
   az role assignment create \
     --role "Virtual Machine Contributor" \
     --assignee "$USER_ASSIGNED_CLIENT_ID" \
     --scope "/subscriptions/${AZ_SUBSCR_ID}/resourcegroups/${AZURE_RESOURCE_GROUP}"
   ```

2. Assign the *"Reader"* role to the user assigned identity:

   ```bash
   az role assignment create \
     --role "Reader" \
     --assignee "$USER_ASSIGNED_CLIENT_ID" \
     --scope "/subscriptions/${AZ_SUBSCR_ID}/resourcegroups/${AZURE_RESOURCE_GROUP}"
   ```

3. Assign the *"Network Contributor"* role to the user assigned identity:

   ```bash
   az role assignment create \
     --role "Network Contributor" \
     --assignee "$USER_ASSIGNED_CLIENT_ID" \
     --scope "/subscriptions/${AZ_SUBSCR_ID}/resourcegroups/${AKS_RG}"
   ```

Create the federated credential for the CAA ServiceAccount using the OIDC endpoint from the AKS cluster:

1. Set the `AKS_OIDC_ISSUER` environment variable to the OIDC issuer URL of your AKS cluster:

   ```bash
   export AKS_OIDC_ISSUER="$(az aks show \
     --name "${CLUSTER_NAME}" \
     --resource-group "${AZURE_RESOURCE_GROUP}" \
     --query "oidcIssuerProfile.issuerUrl" \
     -o tsv)"
   ```

2. Create the federated credential:

   ```bash
   az identity federated-credential create \
     --name "${CLUSTER_NAME}-federated" \
     --identity-name "${AZURE_WORKLOAD_IDENTITY_NAME}" \
     --resource-group "${AZURE_RESOURCE_GROUP}" \
     --issuer "${AKS_OIDC_ISSUER}" \
     --subject system:serviceaccount:confidential-containers-system:cloud-api-adaptor \
     --audience api://AzureADTokenExchange
   ```

## Deploy the CAA Helm chart

> **Note**: If you are using Calico Container Network Interface (CNI) on the Kubernetes cluster, then, [configure](https://projectcalico.docs.tigera.io/networking/vxlan-ipip#configure-vxlan-encapsulation-for-all-inter-workload-traffic) Virtual Extensible LAN (VXLAN) encapsulation for all inter workload traffic.

### Download the CAA Helm chart

Currently, there are three options to choose from when downloading the CAA Helm chart:

- using the latest release,
- using the latest build from the main branch,
- using your own helm build.

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

```bash
CAA_VERSION="$(
  curl -fsSL \
    "https://api.github.com/repos/confidential-containers/cloud-api-adaptor/releases/latest" |
    jq -er '.tag_name | sub("^v"; "")'
)"
curl -LO "https://github.com/confidential-containers/cloud-api-adaptor/archive/refs/tags/v${CAA_VERSION}.tar.gz"
tar -xvzf "v${CAA_VERSION}.tar.gz"
cd "cloud-api-adaptor-${CAA_VERSION}/src/cloud-api-adaptor/install/charts/peerpods"
```

{{% /tab %}}

{{% tab header="Latest Build" %}}

```bash
export CAA_BRANCH="main"
curl -LO "https://github.com/confidential-containers/cloud-api-adaptor/archive/refs/heads/${CAA_BRANCH}.tar.gz"
tar -xvzf "${CAA_BRANCH}.tar.gz"
cd "cloud-api-adaptor-${CAA_BRANCH}/src/cloud-api-adaptor/install/charts/peerpods"
```

{{% /tab %}}

{{% tab header="DIY" %}}
This assumes that you already have the code ready to use. 
On your terminal change directory to the Cloud API Adaptor's code base.
{{% /tab %}}

{{< /tabpane >}}

### Export PodVM image version

Exports the PodVM image ID used by peer pods. This variable tells the deployment tooling which PodVM image version
to use when creating peer pod virtual machines in Azure.

The image is pulled from the Coco community gallery (or manually built) and must match the current CAA release version.

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

Export this environment variable to use for the peer pod VM:

```bash
export AZURE_IMAGE_ID="/CommunityGalleries/cococommunity-42d8482d-92cd-415b-b332-7648bd978eff/Images/peerpod-podvm-fedora/Versions/${CAA_VERSION}"
```

{{% /tab %}}

{{% tab header="Latest Build" %}}

An automated job builds the pod VM image each night at 00:00 UTC. You can use that image by exporting the following environment variable:

```bash
SUCCESS_TIME=$(curl -s \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/confidential-containers/cloud-api-adaptor/actions/workflows/azure-nightly-build.yml/runs?status=success" \
  | jq -r '.workflow_runs[0].updated_at')

export AZURE_IMAGE_ID="/CommunityGalleries/cocopodvm-d0e4f35f-5530-4b9c-8596-112487cdea85/Images/podvm_image0/Versions/$(date -u -jf "%Y-%m-%dT%H:%M:%SZ" "$SUCCESS_TIME" "+%Y.%m.%d" 2>/dev/null || date -d "$SUCCESS_TIME" +%Y.%m.%d)"
```

Above image version is in the format `YYYY.MM.DD`, so to use the latest image should be today's date or yesterday's date.

{{% /tab %}}

{{% tab header="DIY" %}}

To build custom image for the peer pods, follow the steps below:

##### Pre-requisites

This section describes the prerequisites that we assume for the following steps regarding installed software and access to Azure.

Install Required Tools:

- Install [Docker](https://docs.docker.com/engine/install/) with `buildx`
- Install packages:
   - `make`
   - `qemu-utils`
   - `git`
- Install `yq`:
  ```bash
  ARCH=amd64
  sudo curl -fsSL -o /usr/local/bin/yq "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_${ARCH}"
  sudo chmod +x /usr/local/bin/yq
  ```
- Install `az` CLI [tool](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

Clone repository: [Cloud API Adaptor repository](https://github.com/confidential-containers/cloud-api-adaptor.git).

This repository contains the necessary scripts and configurations to build the PodVM image.

##### Build the PodVM image

1. Navigate to the `cloud-api-adaptor/src/cloud-api-adaptor/podvm` directory.

2. Build the **binaries** using below command:

   ```bash
   ARCH=amd64 TEE_PLATFORM=az-cvm-vtpm \
     make podvm-binaries
   ```

   The `ARCH` parameter can be:
   - `amd64` / `x86_64`: 64-bit x86 systems using Intel® or AMD processors
   - `arm64` / `aarch64`: 64-bit Arm systems
   - `s390x`: 64-bit IBM systems
   - `ppc64le`: 64-bit IBM Power systems

   The `TEE_PLATFORM` parameter can be:
   - `none`: for tests with non-confidential guests
   - `all`: for all following platforms
   - `fs`: for platforms with encrypted root filesystems (i.e. s390x)
   - `tdx`: for Intel® TDX
   - `az-tdx-vtpm`: for Intel® TDX with Azure vTPM
   - `snp`/`amd`: for AMD SEV-SNP
   - `az-snp-vtpm`: for AMD SEV-SNP with Azure vTPM
   - `se`: for IBM Secure Execution (SE)

3. Build the **image**:

   - Run below command to build the **release** image:
   
     ```bash
     make image
     ```
   
     > **Note**: This will only build the pod VM image **without** SSH access.
   
   - Run command to build **debug** image
   
     ```bash
     make image-debug
     ```
   
     > **Note**: This will only build the pod VM image **with** SSH access.

4. Convert QCOW2 image to Virtual Hard Disk (VHD) format

   > **Caution**: The below command can produce a VHD that is not properly aligned for Azure, which can cause issues when uploading and using the image.<br>
   To ensure that VHD image is properly aligned, follow the steps below to resize the QCOW2 image to a MiB boundary before converting it to VHD format.

   1. Set the `QCOW2` environment variable to the path of the generated QCOW2 image:

      ```bash
      export QCOW2="build/podvm-fedora-amd64.qcow2"
      ```

   2. Resize qcow2 to MiB boundary:

      ```bash
      TARGET_BYTES=$((900 * 1024 * 1024))
      ALIGNED_SIZE=$(( (TARGET_BYTES + 1024*1024 - 1) / (1024*1024) * (1024*1024) ))
      qemu-img resize "${QCOW2}" "${ALIGNED_SIZE}"
      ```

      Azure requires VHD images to be aligned to 1 MiB, so the QCOW2 image must be resized to a multiple of 1 MiB before conversion.

      > **Note**: Currently the release image size is ~866MB in release builds, so resizing to 900MB ensures that the image is properly aligned while minimizing the additional space used.

   3. Convert to Azure-compatible fixed VHD format:

      ```bash
      qemu-img convert -f qcow2 -O vpc -o subformat=fixed,force_size \
      "${QCOW2}" podvm.vhd
      ```

##### Upload PodVM image

1. Prepare environment variables for the upload process by running the following commands:

   ```bash
   export AZURE_RESOURCE_GROUP="${AZURE_RESOURCE_GROUP}"
   export AZURE_REGION="eastus"
   ```

   > **Note**: Make sure that the `AZURE_REGION` and `AZURE_RESOURCE_GROUP` variables are set to the same region where your AKS cluster is deployed.

2. Create a shared image gallery by running the following command:

   ```bash
   export GALLERY_NAME="caacvmsGallery"
   
   az sig create \
     --gallery-name "${GALLERY_NAME}" \
     --resource-group "${AZURE_RESOURCE_GROUP}" \
     --location "${AZURE_REGION}"
   ```

3. Create the `Image Definition` by running the following command:

   ```bash
   export GALLERY_IMAGE_DEF_NAME="cc-image"
   
   az sig image-definition create \
     --resource-group "${AZURE_RESOURCE_GROUP}" \
     --gallery-name "${GALLERY_NAME}" \
     --gallery-image-definition "${GALLERY_IMAGE_DEF_NAME}" \
     --publisher GreatPublisher \
     --offer GreatOffer \
     --sku GreatSku \
     --os-type "Linux" \
     --os-state "Generalized" \
     --hyper-v-generation "V2" \
     --location "${AZURE_REGION}" \
     --architecture "x64" \
     --features SecurityType=ConfidentialVmSupported
   ```

   > **Note**: The flag `--features SecurityType=ConfidentialVmSupported` allows you to an upload custom image and boot it up as a Confidential Virtual Machine (CVM).

4. Create Storage Account:

   ```bash
   export AZURE_STORAGE_ACCOUNT="coco-storage"
   
   az storage account create \
     --name $AZURE_STORAGE_ACCOUNT \
     --resource-group $AZURE_RESOURCE_GROUP \
     --location $AZURE_REGION \
     --sku Standard_ZRS \
     --encryption-services blob \
     --allow-blob-public-access false
   ```

5. Create storage container:

   ```bash
   export AZURE_STORAGE_CONTAINER=vhd
   
   az storage container create \
     --account-name $AZURE_STORAGE_ACCOUNT \
     --name $AZURE_STORAGE_CONTAINER \
     --auth-mode login \
     --public-access off
   ```

6. Get storage key:

   ```bash
   AZURE_STORAGE_KEY=$(az storage account keys list \
                         --resource-group $AZURE_RESOURCE_GROUP \
                         --account-name $AZURE_STORAGE_ACCOUNT \
                         --query "[?keyName=='key1'].{Value:value}" \
                         --output tsv)
   echo $AZURE_STORAGE_KEY
   ```

7. Upload VHD file to Azure Storage:

   ```bash
   BLOB_NAME="podvm.vhd"
    
   az storage blob upload \
     --account-name "$AZURE_STORAGE_ACCOUNT" \
     --container-name "$AZURE_STORAGE_CONTAINER" \
     --name "$BLOB_NAME" \
     --auth-mode key \
     --account-key "$AZURE_STORAGE_KEY" \
     --file podvm.vhd
   ```

8. Set the `AZURE_STORAGE_EP` environment variable to the blob service endpoint of the storage account:

   ```bash
   AZURE_STORAGE_EP=$(az storage account list \
                        -g $AZURE_RESOURCE_GROUP \
                        --query "[].{uri:primaryEndpoints.blob} | [? contains(uri, '$AZURE_STORAGE_ACCOUNT')]" \
                        --output tsv)
   echo $AZURE_STORAGE_EP
   ```

9. Set the `VHD_URI` environment variable to the URI of the uploaded VHD file:

   ```bash
   export VHD_URI="${AZURE_STORAGE_EP}${AZURE_STORAGE_CONTAINER}/${BLOB_NAME}"
   echo $VHD_URI
   ```

10. Create a managed image from the VHD:

    ```bash
    export MANAGED_IMAGE_NAME="cc-podvm-image"

    az image create \
      --resource-group "$AZURE_RESOURCE_GROUP" \
      --name "$MANAGED_IMAGE_NAME" \
      --location "$AZURE_REGION" \
      --os-type Linux \
      --hyper-v-generation V2 \
      --source "$VHD_URI"
    ```

11. Set the `MANAGED_IMAGE_ID` environment variable to the ID of the created managed image:

    ```bash
    MANAGED_IMAGE_ID=$(az image show \
                         --resource-group "$AZURE_RESOURCE_GROUP" \
                         --name "$MANAGED_IMAGE_NAME" \
                         --query "id" \
                         --output tsv)
    echo "$MANAGED_IMAGE_ID"
    ```

12. Create an image version in the Shared Image Gallery using the managed image:

    ```bash
    az sig image-version create \
      --resource-group "$AZURE_RESOURCE_GROUP" \
      --gallery-name "$GALLERY_NAME"  \
      --gallery-image-definition "$GALLERY_IMAGE_DEF_NAME" \
      --gallery-image-version "1.0.0" \
      --target-regions "$AZURE_REGION" \
      --managed-image "$MANAGED_IMAGE_ID"
    ```

13. Retrieve the image name and export it to use in CAA configuration:

    ```bash
    export AZURE_IMAGE_ID=$(az sig image-version list \
                       --resource-group "$AZURE_RESOURCE_GROUP" \
                       --gallery-name "$GALLERY_NAME" \
                       --gallery-image-definition "$GALLERY_IMAGE_DEF_NAME" \
                       --query "sort_by([], &name)[-1].id" \
                       --output tsv)
    echo "$AZURE_IMAGE_ID"
    ```

{{% /tab %}}

{{< /tabpane >}}

### Set the CAA container image and tag

Define the Cloud API Adaptor (CAA) container image to deploy.
These variables tell the deployment tooling which CAA image and architecture-specific tag to pull and run.
The tag is derived from the CAA release version to ensure compatibility with the selected PodVM image and configuration.

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

Export the following environment variable to use the latest release image of CAA:

```bash
export CAA_IMAGE="quay.io/confidential-containers/cloud-api-adaptor"
export CAA_TAG="v${CAA_VERSION}-amd64"
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

> **Caution**: You can also use the `latest` tag, but it is **not** recommended, 
> because of its lack of version control and potential for unpredictable 
> updates, impacting stability and reproducibility in deployments.

{{% /tab %}}

{{% tab header="DIY" %}}

If you have made changes to the CAA code and you want to deploy those changes then follow [these instructions](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/README.md#building-custom-cloud-api-adaptor-image) to build the container image. Once the image is built export the environment variables `CAA_IMAGE` and `CAA_TAG`.

{{% /tab %}}

{{< /tabpane >}}

### Select peer-pods machine type

{{< tabpane text=true right=true persist=header >}}
{{% tab header="AMD SEV-SNP" %}}

```bash
export AZURE_INSTANCE_SIZE="Standard_DC2as_v5"
export DISABLECVM="false"
```

Find more AMD SEV-SNP machine types on [this](https://learn.microsoft.com/en-us/azure/virtual-machines/dasv5-dadsv5-series) Azure documentation.

{{% /tab %}}

{{% tab header="Intel® TDX" %}}

```bash
export AZURE_INSTANCE_SIZE="Standard_DC2es_v6"
export DISABLECVM="false"
```

Find more Intel® TDX machine types on [this](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dcesv6-series?tabs=sizebasic) Azure documentation.

{{% /tab %}}

{{% tab header="Non-Confidential" %}}

```bash
export AZURE_INSTANCE_SIZE="Standard_D2as_v5"
export DISABLECVM="true"
```

{{% /tab %}}
{{< /tabpane >}}

### Populate the provider file

List of all available configuration options can be found in two places:

- [Main charts values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/values.yaml)
- [Azure specific values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/azure.yaml)

Run the following command to update the [`providers/azure.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/azure.yaml) file:

```bash
cat <<EOF > providers/azure.yaml
provider: azure
image:
  name: "${CAA_IMAGE}"
  tag: "${CAA_TAG}"
providerConfigs:
   azure:
      AZURE_IMAGE_ID: "${AZURE_IMAGE_ID}"
      AZURE_REGION: "${AZURE_REGION}"
      AZURE_RESOURCE_GROUP: "${AZURE_RESOURCE_GROUP}"
      AZURE_SUBNET_ID: "${AZURE_SUBNET_ID}"
      AZ_SUBSCR_ID: "${AZ_SUBSCR_ID}"
      AZURE_INSTANCE_SIZE: "${AZURE_INSTANCE_SIZE}"
      DISABLECVM: ${DISABLECVM}
EOF
```

### Deploy helm chart

1. Create file `namespace.yaml` with the following content:

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: confidential-containers-system
     labels:
       app.kubernetes.io/managed-by: Helm
     annotations:
       meta.helm.sh/release-name: peerpods
       meta.helm.sh/release-namespace: confidential-containers-system
   ```

   This namespace will be used to deploy CAA and related components, and it is labeled and annotated to be managed by Helm.

2. Create namespace managed by Helm:

   ```bash
   kubectl apply -f namespace.yaml
   ```

3. Create a Kubernetes Secret that stores the Azure service-account credentials:

   See [providers/azure-secrets.yaml.template](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/azure-secrets.yaml.template) for required keys.

   > **Note**: Below example assumes that you are using workload identity for authentication hence
   > `AZURE_CLIENT_SECRET` and `AZURE_TENANT_ID` are not provided.

   ```bash
   kubectl create secret generic my-provider-creds \
     -n confidential-containers-system \
     --from-literal=AZURE_CLIENT_ID="${USER_ASSIGNED_CLIENT_ID}" \
     --from-file=id_rsa.pub=${SSH_KEY}
   ```

   > **Note**: `--from-file=id_rsa.pub=${SSH_KEY}` is optional. It allows user to SSH into the pod VMs for troubleshooting purposes.
   > This option works only for custom debug enabled pod VM images. The prebuilt pod VM images do not have SSH connection enabled.

4. Install helm chart:

   Below command uses customization options `-f` and `--set` which are described [here](../../getting-started/installation/advanced_configuration).

    ```bash
    helm install peerpods . \
      -f providers/azure.yaml \
      --set secrets.mode=reference \
      --set secrets.existingSecretName=my-provider-creds \
      --set-json daemonset.podLabels='{"azure.workload.identity/use":"true"}' \
      --dependency-update \
      -n confidential-containers-system
    ```

    > **Note**: Above example assumes that you are using workload identity for authentication. <br>
    > This line: `--set-json daemonset.podLabels='{"azure.workload.identity/use":"true"}'` is required **only** when using workload identity.

Generic Peer pods Helm charts deployment instructions are also described
[here](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/install/charts/peerpods/README.md).

### Verify deployment

Verify that the `runtimeclass` is created after deploying Peer Pods Helm Charts:

```bash
kubectl get runtimeclass
```

Once you can find a `runtimeclass` named `kata-remote` then you can be sure that the deployment was successful.
A successful deployment will look like this:

```console
$ kubectl get runtimeclass
NAME          HANDLER       AGE
kata-remote   kata-remote   7m18s
```

## Run sample application

To verify that everything is working as expected, you can deploy a sample application using the `kata-remote` runtime class provided by CAA.

1. Create a deployment file `nginx-deployment.yaml` for a sample nginx application:

   ```yaml
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
   ```

2. Apply created configuration to the cluster

   ```bash
   kubectl apply -f nginx-deployment.yaml
   ```

3. Ensure that the pod is up and running:

   ```bash
   kubectl get pods -n default
   ```

4. Verify that the PodVM was created by running the following command:

   ```bash
   az vm list \
     --resource-group "${AZURE_RESOURCE_GROUP}" \
     --output table
   ```

   In command output there should be the VM associated with the pod `nginx`.


> **Note**: If you run into problems then check the troubleshooting guide [here](../troubleshooting/).

## Pod VM reference values

Reference values for the PCRs can be retrieved from the registry and used in attestation policies to verify the integrity of the Pod VM image before trusting it with sensitive workloads or secrets.

As part of a Pod VM image build expected PCR measurements are published into an OCI registry.

### Pre-requisites

Install [ORAS](https://oras.land/) tool to pull the reference values from the OCI registry and [GitHub CLI](https://cli.github.com/) to verify its build provenance.

### Verify build provenance

Assert that the measurements have been generated by a trusted build process on the official repository. 
Specify `--format=json` to get more details about the build process.

```bash
CAA_REPO="confidential-containers/cloud-api-adaptor"
OCI_REGISTRY="ghcr.io/${CAA_REPO}/measurements/azure/podvm:${CAA_VERSION}"
gh attestation verify -R "$CAA_REPO" "oci://${OCI_REGISTRY}"
```

### Retrieve reference values

The PCR values can be used in remote attestation policies to assert the integrity of the PodVM image.

```console
$ oras pull "$OCI_REGISTRY"
$ jq -r .measurements.sha256.pcr11 < measurements.json
0x58e8afdf5b105fc6b202eb8e537a9f1512a4b33cd5921171b518645a86ca5a75
```

## Uninstall

To uninstall Confidential Containers from Azure AKS cluster, use the following commands:

1. Remove all pods with `kata-runtime` runtime class:

    ```bash
    kubectl get pods -A -o custom-columns='NAME:.metadata.name,NAMESPACE:.metadata.namespace,RUNTIMECLASS:.spec.runtimeClassName' \
    | grep kata-remote \
    | awk '{print $1, $2}' \
    | xargs -n 2 sh -c 'kubectl delete pod -n "$2" "$1"' _
    ```

2. List deployed Confidential Containers Helm chart:

   > **Note**: This command assumes that only one Helm release is deployed in the `confidential-containers-system` namespace. 
   > If there are multiple releases, you may need to adjust the command to select the correct one.

   ```bash
   export HELM_COCO_CHART_NAME=$(helm list \
                                  -n confidential-containers-system \
                                  --short)
   ```

3. Delete Confidential Containers related Helm chart:

   ```bash
   helm uninstall ${HELM_COCO_CHART_NAME} \
     --namespace confidential-containers-system
   ```

4. Delete secret with provider credentials `my-provider-creds`:

   ```bash
   kubectl delete secret my-provider-creds \
     -n confidential-containers-system
   ```

5. Delete Confidential Containers related namespace:

   ```bash
   kubectl delete namespace confidential-containers-system
   ```

6. Delete the AKS cluster by running the following command:

   ```bash
   az group delete \
     --name "${AZURE_RESOURCE_GROUP}" \
     --yes --no-wait
   ```
