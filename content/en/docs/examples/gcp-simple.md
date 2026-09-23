---
title: GCP
description: Peer Pods Helm Chart using Cloud API Adaptor (CAA) on GCP
categories:
  - examples
tags:
  - helm
  - caa
  - gcp
  - gke
---

This documentation will walk you through setting up Cloud API Adaptor (CAA) (a.k.a. Peer Pods) on 
Google Kubernetes Engine (GKE). 

It explains how to deploy:

- A single worker node Kubernetes cluster using Google Kubernetes Engine (GKE),
- CAA on that Kubernetes cluster,
- A sample application deployed using CAA to verify that everything is working as expected.

## Pre-requisites

Install Required Tools:

- Install [kubectl](https://kubernetes.io/docs/tasks/tools/),
- Install [Helm](https://helm.sh/docs/intro/install),
- Install `gcloud` CLI [tool](https://cloud.google.com/sdk/docs/install).

Google Cloud Project:

- Ensure you have a Google Cloud project created,
- Note the Project ID (export it as `GCP_PROJECT_ID`).

## GCP Preparation

1. Set the environment variable `GCP_PROJECT_ID` to your Google Cloud project ID:

    ```bash
    export GCP_PROJECT_ID="YOUR_PROJECT_ID"
    ```

2. Authenticate with Google Cloud and set the project:

    ```bash
    gcloud auth login
    gcloud config set project "${GCP_PROJECT_ID}"
    ```

3. Enable the GKE API:

    ```bash
    gcloud services enable container.googleapis.com \
      --project="${GCP_PROJECT_ID}"
    ```

   This API is required to create and manage the GKE cluster.

4. Set the `GCP_REGION` environment variable to the desired region for your GKE cluster with Intel® TDX supported instances:

    ```bash
    export GCP_REGION="us-central1"
    ```

    {{% alert title="Note" color="primary" %}}
    "us-central1" was chosen because supports Confidential VMs.<br> 
    For a complete list of supported regions visit [supported-configurations](https://cloud.google.com/confidential-computing/confidential-vm/docs/supported-configurations#supported-zones).
    {{% /alert %}}

## Deploy Kubernetes Using GKE

Deploy a single node Kubernetes cluster using GKE:

```bash
export GKE_CLUSTER_NAME="caa-gke"

gcloud container clusters create "${GKE_CLUSTER_NAME}" \
  --zone ${GCP_REGION}-a \
  --machine-type "e2-standard-4" \
  --image-type UBUNTU_CONTAINERD \
  --num-nodes 1
```

> **Note**: The `e2-standard-4` machine type is used for the GKE cluster nodes, which is a general-purpose machine.<br>
The `UBUNTU_CONTAINERD` image type is specified to ensure compatibility with the container runtime used by CAA.

Get cluster credentials:

```bash
gcloud container clusters get-credentials "${GKE_CLUSTER_NAME}" \
  --zone "${GCP_REGION}-a" \
  --project "${GCP_PROJECT_ID}"
```

**(Optional)** Verify that the cluster is reachable:

```bash
kubectl get nodes -o wide
```

Label the worker node:

```bash
kubectl get nodes \
  --selector='!node-role.kubernetes.io/master' \
  -o name \
  | xargs -I{} kubectl label {} node.kubernetes.io/worker=
```

This labeling step adds the `node.kubernetes.io/worker` label to non-control-plane nodes.

{{% alert title="Note" color="primary" %}}
Starting with GKE version 1.27, GCP configures containerd with the `discard_unpacked_layers=true` flag to optimize disk
usage by removing compressed image layers after they are unpacked. However, this can cause issues with PeerPods,
as the workload may fail to locate required layers.

To avoid this, disable the `discard_unpacked_layers` setting in the containerd configuration.

If you encounter problem with VM's not running check [Troubleshooting](#troubleshooting) section on this page.

{{% /alert %}}

## Configure VPC network

We need to make sure port 15150 is open under the default VPC network:

```bash
gcloud compute firewall-rules create allow-port-15150 \
    --project=${GCP_PROJECT_ID} \
    --network=default \
    --allow=tcp:15150
```

For production scenarios, it is advisable to restrict the source IP range to
minimize security risks. For example, you can restrict the source range to a
specific IP address or CIDR block:

```bash
gcloud compute firewall-rules create allow-port-15150-restricted \
   --project=${GCP_PROJECT_ID} \
   --network=default \
   --allow=tcp:15150 \
   --source-ranges=[YOUR_EXTERNAL_IP]
```

## Setup CAA Requirements

### Enable Additional APIs

Enable the Compute Engine and IAM APIs required for CAA:

```bash
gcloud services enable compute.googleapis.com iam.googleapis.com \
  --project="${GCP_PROJECT_ID}"
```

These APIs are required to:

- provision the confidential PodVM instances and related networking resources,
- create and authorize the service account that Cloud API Adaptor uses to access GCP.

### Create Service Account and Credentials

1. Create a service account for peer pods and grant it the required permissions:

   ```bash
   gcloud iam service-accounts create peerpods \
     --description="Peerpods Service Account" \
     --display-name="Peerpods Service Account"

   gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
     --member="serviceAccount:peerpods@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
     --role="roles/compute.instanceAdmin.v1"

   gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
     --member="serviceAccount:peerpods@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
     --role="roles/iam.serviceAccountUser"
   ```

   These roles allow the Cloud API Adaptor to:

   - create, start/stop, and delete the Compute Engine instances used as **PodVMs** (`roles/compute.instanceAdmin.v1`),
   - run actions as the `peerpods` service account when provisioning those resources (service-account impersonation via `roles/iam.serviceAccountUser`).

   > **Note**: IAM policy updates can take a few minutes to propagate. If later steps fail with permission errors, wait briefly and retry.

2. Set the `GOOGLE_APP_CREDENTIALS` environment variable to point to the credentials file that will be generated in the next step:

    ```bash
    export GOOGLE_APP_CREDENTIALS=~/.config/gcloud/peerpods_application_key.json
    ```

3. Generate and save the credentials file:

    ```bash
    gcloud iam service-accounts keys create \
      "${GOOGLE_APP_CREDENTIALS}" \
      --iam-account="peerpods@${GCP_PROJECT_ID}.iam.gserviceaccount.com"
    ```

## Build and publish the PodVM image

### Pre-requisites

This section describes the prerequisites that we assume for the following steps regarding installed software and access to Google Cloud.

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
- Install `gcloud` CLI [tool](https://cloud.google.com/sdk/docs/install)

Clone repository: [Cloud API Adaptor repository](https://github.com/confidential-containers/cloud-api-adaptor.git).

This repository contains the necessary scripts and configurations to build the PodVM image.

### Build the PodVM image

1. Navigate to the `cloud-api-adaptor/src/cloud-api-adaptor/podvm` directory.

2. Build the **binaries** using below command:

   ```bash
   ARCH=amd64 TEE_PLATFORM=tdx \
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

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Build types**:" disabled=true /%}}

{{% tab header="Release" %}}

Run below command to build the release image:

```bash
make image
```

> **Note**: This will only build the pod VM image **without** SSH access.

{{% /tab %}}

{{% tab header="Debug" %}}

1. Prepare SSH key to build debug image

   For using SSH, create a file `resources/authorized_keys` with your SSH public key.
   Ensure the permissions are set to `0400` for the `authorized_keys` file.
   SSH access is only possible for the `root` user.

   Below are the commands to generate a new SSH key and create the `authorized_keys` file:

   1. Create SSH key pair and copy public keys to proper location:

      ```bash
      ssh-keygen -t rsa -f ./gcp_ssh_debug -C gcp_ssh_debug
      cp ./gcp_ssh_debug.pub resources/authorized_keys
      chmod 400 resources/authorized_keys
      ```

   2. Add credentials to google using CLI

       ```bash
       gcloud compute os-login ssh-keys add \
       --key-file=$(realpath ./gcp_ssh_debug.pub) \
       --project=${GCP_PROJECT_ID} \
       --ttl=0
       ```

      > **Note**: TTL (time to live) is set to 0, which means that the key will not expire. 
      > You can set it to any value you want, for example `1h` for 1 hour or `30m` for 30 minutes.

2. Run command to build debug image
    
   ```bash
   make image-debug
   ```

   > **Note**: This will only build the pod VM image **with** SSH access.

{{% /tab %}}

{{< /tabpane >}} 

Above commands will produce `./build/system.raw` (~1.6GB), a disk image that can be booted with an ESP/UEFI partition.

### Publish image to Google Storage

1. Prepare the raw disk image and package it as `build/disk.tar.gz`:

   ```bash
   cp build/system.raw build/disk.raw && \
     tar -cvzf build/disk.tar.gz -C build disk.raw
   ```

2. Export the following environment variables:

   ```bash
   export GCP_PROJECT_ID="YOUR_PROJECT_ID"
   export GCP_REGION="us-central1"
   export BUCKET_NAME="peerpods-bucket"
   ```

   > **Note**: Above values should be set according to your Google Cloud project and region set in previous steps.<br>
   > The `BUCKET_NAME` should be globally unique across all of Google Cloud, so consider adding a random suffix if needed.

3. Login to Google account and follow instructions in command line to authenticate:

    ```bash
    gcloud init
    ```

4. Create a GCS bucket for images:

   ```bash
   gcloud storage buckets create "gs://${BUCKET_NAME}" \
     --project="${GCP_PROJECT_ID}" \
     --location="${GCP_REGION}"
   ```

5. Upload the disk image to a bucket and create the image:

   1. Prepare image name:

      ```bash
      export IMAGE_BASE_NAME="podvm-image"
      export CAA_HASH=$(git rev-parse --short HEAD)
      export IMAGE_NAME="${IMAGE_BASE_NAME}-${CAA_HASH}-release"
      ```

      > **Note**: For consistency, the git commit hash is part of image name and release type (debug/release) to differentiate between development and production builds.

   2. Upload image to GCS bucket:
      ```bash
      gcloud storage cp build/disk.tar.gz gs://${BUCKET_NAME}/peerpods-disk.tar.gz
      ```

   3. Create image in GCP with defined name from the uploaded disk image:

      {{< tabpane text=true right=true persist=header >}}
      
   {{% tab header="AMD SEV-SNP" %}}
   ```bash
   gcloud compute images create ${IMAGE_NAME} \
     --source-uri=gs://${BUCKET_NAME}/peerpods-disk.tar.gz \
     --guest-os-features=UEFI_COMPATIBLE
   ```

   This command creates a new image in GCP with the specified name and the uploaded disk image.<br>
   The `--guest-os-features` flag ensures that the image is compatible with UEFI.
   {{% /tab %}}
      
   {{% tab header="Intel® TDX" %}}

   ```bash
   gcloud compute images create ${IMAGE_NAME} \
     --source-uri=gs://${BUCKET_NAME}/peerpods-disk.tar.gz \
     --guest-os-features=UEFI_COMPATIBLE,TDX_CAPABLE
   ```

   **Both** `UEFI_COMPATIBLE` and `TDX_CAPABLE` are required for `tdx` TEE.

   This command creates a new image in GCP with the specified name and the uploaded disk image.<br>
   The `--guest-os-features` flag ensures that the image is compatible with UEFI and TDX.

   {{% /tab %}}
       
   {{< /tabpane >}}

### Export PodVM image id

Export the PodVM image id to be used in the provider configuration. This is the name of the image created in the previous step.

```bash
export PODVM_IMAGE_ID="podvm-image-00754585-release"
```

<details>
   <summary><strong>Show command how to retrieve latest published image from GCP</strong></summary>

   Run below command to retrieve the latest published image from GCP:

   ```bash
   gcloud compute images list \
     --project="${GCP_PROJECT_ID}" \
     --filter="name ~ ^${IMAGE_BASE_NAME}-" \
     --sort-by=~creationTimestamp \
     --limit=1 \
     --format="value(name)"
   ```

   </details>

## Deploy

### Install cert-manager

The Peer Pods Helm chart requires cert-manager for webhook certificates. Install it first:

```bash
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager with CRDs
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.19.1 \
  --set crds.enabled=true
```

Wait for cert-manager to be ready:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=300s
```

### Set TEE Platform Configuration

Set TEE platform and PodVM instance type for your workload:

{{< tabpane text=true right=true persist=header >}}

{{% tab header="AMD SEV-SNP" %}}
```bash
export PODVM_INSTANCE_TYPE="n2d-standard-4"
export DISABLECVM=false
export GCP_CONFIDENTIAL_TYPE="SEV" # SEV or SEV_SNP
export GCP_DISK_TYPE="pd-standard"
```
{{% /tab %}}

{{% tab header="Intel® TDX" %}}
```bash
export PODVM_INSTANCE_TYPE="c3-standard-4"
export DISABLECVM=false
export GCP_CONFIDENTIAL_TYPE="TDX"
export GCP_DISK_TYPE="pd-balanced"
```

For the purposes of this example, we use a C3 machine type that supports Intel® TDX.

> **Note**: Choose a C3 machine type that fits your workload from the list of supported options in the [Google Cloud C3 machine types documentation](https://docs.cloud.google.com/compute/docs/general-purpose-machines#c3_machine_types).

{{% /tab %}}

{{% tab header="Non-Confidential" %}}
```bash
export PODVM_INSTANCE_TYPE="e2-medium"
export DISABLECVM=true
export GCP_CONFIDENTIAL_TYPE=""
export GCP_DISK_TYPE="pd-standard"
```
{{% /tab %}}

{{< /tabpane >}}

### Download the CAA Helm deployment artifacts

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

If you have made changes to the CAA code and you want to deploy those changes
then follow [these
instructions](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/README.md#building-custom-cloud-api-adaptor-image)
to build the container image. Once the image is built export the environment
variables `CAA_IMAGE` and `CAA_TAG`.

{{% /tab %}}

{{< /tabpane >}}

### Populate the provider file

List of all available configuration options can be found in two places:
- [Main charts values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/values.yaml)
- [GCP specific values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/gcp.yaml)

Run the following command to update the [`providers/gcp.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/gcp.yaml) file:

```bash
cat <<EOF > providers/gcp.yaml
provider: gcp
image:
  name: "${CAA_IMAGE}"
  tag: "${CAA_TAG}"
providerConfigs:
  gcp:
    GCP_NETWORK: "global/networks/default"
    GCP_PROJECT_ID: "${GCP_PROJECT_ID}"
    GCP_ZONE: "${GCP_REGION}-a"
    GCP_MACHINE_TYPE: "${PODVM_INSTANCE_TYPE}"
    GCP_DISK_TYPE: "${GCP_DISK_TYPE}"
    PODVM_IMAGE_NAME: "${PODVM_IMAGE_ID}"
    GCP_CONFIDENTIAL_TYPE: "${GCP_CONFIDENTIAL_TYPE}"
    DISABLECVM: ${DISABLECVM}
EOF
```

### Deploy the CAA Helm chart

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

3. Create a Kubernetes Secret that stores the GCP service-account credentials:

   See [providers/gcp-secrets.yaml.template](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/gcp-secrets.yaml.template) for required keys.

   ```bash
   kubectl create secret generic my-provider-creds \
     -n confidential-containers-system \
     --from-file=GCP_CREDENTIALS="${GOOGLE_APP_CREDENTIALS}"
   ```

   The CAA Helm chart references this secret to authenticate to Google Cloud when provisioning PodVMs.

4. Install helm chart:

   Below command uses customization options `-f` and `--set` which are described [here](../../getting-started/installation/advanced_configuration).

    ```bash
    helm install peerpods . \
      -f providers/gcp.yaml \
      --set secrets.mode=reference \
      --set secrets.existingSecretName=my-provider-creds \
      --dependency-update \
      -n confidential-containers-system
    ```

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

{{< tabpane text=true right=true persist=header >}}

{{% tab header="CoCo Secret Retrieval"  %}}
This example showcases a more advanced deployment using TEE and confidential
VMs with the kata-remote runtime class. It demonstrates how to deploy a sample
pod and retrieve a secret securely within a confidential computing environment.

##### Prepare the init data configuration

Peerpods now supports init data, you can pass the required configuration files
(`aa.toml`, `cdh.toml`, and `policy.rego`) via the
`io.katacontainers.config.hypervisor.cc_init_data` annotation. Below is an example
of the configuration and usage.

```toml
# initdata.toml
algorithm = "sha384"
version = "0.1.0"

[data]
"aa.toml" = '''
[token_configs]
[token_configs.coco_as]
url = 'http://127.0.0.1:8080'

[token_configs.kbs]
url = 'http://127.0.0.1:8080'
cert = """
-----BEGIN CERTIFICATE-----
MIIDljCCAn6gAwIBAgIUR/UNh13GFam4emgludtype/S9BIwDQYJKoZIhvcNAQEL
BQAwdTELMAkGA1UEBhMCQ04xETAPBgNVBAgMCFpoZWppYW5nMREwDwYDVQQHDAhI
YW5nemhvdTERMA8GA1UECgwIQUFTLVRFU1QxFDASBgNVBAsMC0RldmVsb3BtZW50
MRcwFQYDVQQDDA5BQVMtVEVTVC1IVFRQUzAeFw0yNDAzMTgwNzAzNTNaFw0yNTAz
MTgwNzAzNTNaMHUxCzAJBgNVBAYTAkNOMREwDwYDVQQIDAhaaGVqaWFuZzERMA8G
A1UEBwwISGFuZ3pob3UxETAPBgNVBAoMCEFBUy1URVNUMRQwEgYDVQQLDAtEZXZl
bG9wbWVudDEXMBUGA1UEAwwOQUFTLVRFU1QtSFRUUFMwggEiMA0GCSqGSIb3DQEB
AQUAA4IBDwAwggEKAoIBAQDfp1aBr6LiNRBlJUcDGcAbcUCPG6UzywtVIc8+comS
ay//gwz2AkDmFVvqwI4bdp/NUCwSC6ShHzxsrCEiagRKtA3af/ckM7hOkb4S6u/5
ewHHFcL6YOUp+NOH5/dSLrFHLjet0dt4LkyNBPe7mKAyCJXfiX3wb25wIBB0Tfa0
p5VoKzwWeDQBx7aX8TKbG6/FZIiOXGZdl24DGARiqE3XifX7DH9iVZ2V2RL9+3WY
05GETNFPKtcrNwTy8St8/HsWVxjAzGFzf75Lbys9Ff3JMDsg9zQzgcJJzYWisxlY
g3CmnbENP0eoHS4WjQlTUyY0mtnOwodo4Vdf8ZOkU4wJAgMBAAGjHjAcMBoGA1Ud
EQQTMBGCCWxvY2FsaG9zdIcEfwAAATANBgkqhkiG9w0BAQsFAAOCAQEAKW32spii
t2JB7C1IvYpJw5mQ5bhIlldE0iB5rwWvNbuDgPrgfTI4xiX5sumdHw+P2+GU9KXF
nWkFRZ9W/26xFrVgGIS/a07aI7xrlp0Oj+1uO91UhCL3HhME/0tPC6z1iaFeZp8Y
T1tLnafqiGiThFUgvg6PKt86enX60vGaTY7sslRlgbDr9sAi/NDSS7U1PviuC6yo
yJi7BDiRSx7KrMGLscQ+AKKo2RF1MLzlJMa1kIZfvKDBXFzRd61K5IjDRQ4HQhwX
DYEbQvoZIkUTc1gBUWDcAUS5ztbJg9LCb9WVtvUTqTP2lGuNymOvdsuXq+sAZh9b
M9QaC1mzQ/OStg==
-----END CERTIFICATE-----
"""
'''

"cdh.toml"  = '''
socket = 'unix:///run/confidential-containers/cdh.sock'
credentials = []

[kbc]
name = 'cc_kbc'
url = 'http://1.2.3.4:8080'
kbs_cert = """
-----BEGIN CERTIFICATE-----
MIIFTDCCAvugAwIBAgIBADBGBgkqhkiG9w0BAQowOaAPMA0GCWCGSAFlAwQCAgUA
oRwwGgYJKoZIhvcNAQEIMA0GCWCGSAFlAwQCAgUAogMCATCjAwIBATB7MRQwEgYD
VQQLDAtFbmdpbmVlcmluZzELMAkGA1UEBhMCVVMxFDASBgNVBAcMC1NhbnRhIENs
YXJhMQswCQYDVQQIDAJDQTEfMB0GA1UECgwWQWR2YW5jZWQgTWljcm8gRGV2aWNl
czESMBAGA1UEAwwJU0VWLU1pbGFuMB4XDTIzMDEyNDE3NTgyNloXDTMwMDEyNDE3
NTgyNlowejEUMBIGA1UECwwLRW5naW5lZXJpbmcxCzAJBgNVBAYTAlVTMRQwEgYD
VQQHDAtTYW50YSBDbGFyYTELMAkGA1UECAwCQ0ExHzAdBgNVBAoMFkFkdmFuY2Vk
IE1pY3JvIERldmljZXMxETAPBgNVBAMMCFNFVi1WQ0VLMHYwEAYHKoZIzj0CAQYF
K4EEACIDYgAExmG1ZbuoAQK93USRyZQcsyobfbaAEoKEELf/jK39cOVJt1t4s83W
XM3rqIbS7qHUHQw/FGyOvdaEUs5+wwxpCWfDnmJMAQ+ctgZqgDEKh1NqlOuuKcKq
2YAWE5cTH7sHo4IBFjCCARIwEAYJKwYBBAGceAEBBAMCAQAwFwYJKwYBBAGceAEC
BAoWCE1pbGFuLUIwMBEGCisGAQQBnHgBAwEEAwIBAzARBgorBgEEAZx4AQMCBAMC
AQAwEQYKKwYBBAGceAEDBAQDAgEAMBEGCisGAQQBnHgBAwUEAwIBADARBgorBgEE
AZx4AQMGBAMCAQAwEQYKKwYBBAGceAEDBwQDAgEAMBEGCisGAQQBnHgBAwMEAwIB
CDARBgorBgEEAZx4AQMIBAMCAXMwTQYJKwYBBAGceAEEBEDDhCejDUx6+dlvehW5
cmmCWmTLdqI1L/1dGBFdia1HP46MC82aXZKGYSutSq37RCYgWjueT+qCMBE1oXDk
d1JOMEYGCSqGSIb3DQEBCjA5oA8wDQYJYIZIAWUDBAICBQChHDAaBgkqhkiG9w0B
AQgwDQYJYIZIAWUDBAICBQCiAwIBMKMDAgEBA4ICAQACgCai9x8DAWzX/2IelNWm
ituEBSiq9C9eDnBEckQYikAhPasfagnoWFAtKu/ZWTKHi+BMbhKwswBS8W0G1ywi
cUWGlzigI4tdxxf1YBJyCoTSNssSbKmIh5jemBfrvIBo1yEd+e56ZJMdhN8e+xWU
bvovUC2/7Dl76fzAaACLSorZUv5XPJwKXwEOHo7FIcREjoZn+fKjJTnmdXce0LD6
9RHr+r+ceyE79gmK31bI9DYiJoL4LeGdXZ3gMOVDR1OnDos5lOBcV+quJ6JujpgH
d9g3Sa7Du7pusD9Fdap98ocZslRfFjFi//2YdVM4MKbq6IwpYNB+2PCEKNC7SfbO
NgZYJuPZnM/wViES/cP7MZNJ1KUKBI9yh6TmlSsZZOclGJvrOsBZimTXpATjdNMt
cluKwqAUUzYQmU7bf2TMdOXyA9iH5wIpj1kWGE1VuFADTKILkTc6LzLzOWCofLxf
onhTtSDtzIv/uel547GZqq+rVRvmIieEuEvDETwuookfV6qu3D/9KuSr9xiznmEg
xynud/f525jppJMcD/ofbQxUZuGKvb3f3zy+aLxqidoX7gca2Xd9jyUy5Y/83+ZN
bz4PZx81UJzXVI9ABEh8/xilATh1ZxOePTBJjN7lgr0lXtKYjV/43yyxgUYrXNZS
oLSG2dLCK9mjjraPjau34Q==
-----END CERTIFICATE-----
"""
'''

"policy.rego" = '''
package agent_policy

import future.keywords.in
import future.keywords.every

import input

# Default values, returned by OPA when rules cannot be evaluated to true.
default CopyFileRequest := true
default CreateContainerRequest := true
default CreateSandboxRequest := true
default DestroySandboxRequest := true
default ExecProcessRequest := false
default GetOOMEventRequest := true
default GuestDetailsRequest := true
default OnlineCPUMemRequest := true
default PullImageRequest := true
default ReadStreamRequest := false
default RemoveContainerRequest := true
default RemoveStaleVirtiofsShareMountsRequest := true
default SignalProcessRequest := true
default StartContainerRequest := true
default StatsContainerRequest := true
default TtyWinResizeRequest := true
default UpdateEphemeralMountsRequest := true
default UpdateInterfaceRequest := true
default UpdateRoutesRequest := true
default WaitProcessRequest := true
default WriteStreamRequest := false
'''
```

Make sure you have the right policy and KBC URL is pointing to your Key Broker Service.

Now, encode the `initdata.toml` and store it in a variable

```bash
INITDATA=$(cat initdata.toml | gzip | base64 -w0)
```

Deploy the pod with:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  annotations:
    io.katacontainers.config.hypervisor.cc_init_data: "$INITDATA"
spec:
  runtimeClassName: kata-remote
  containers:
    - name: example-container
      image: alpine:latest
      command:
        - sleep
        - "3600"
      securityContext:
        privileged: false
        seccompProfile:
          type: RuntimeDefault
EOF
```

##### Fetching Secrets from Trustee

Once the pod is successfully deployed with the `initdata`, you can retrieve secrets from the Trustee service running inside the pod. 
Use the following command to fetch a specific secret:

```bash
kubectl exec -it example-pod -- curl http://127.0.0.1:8006/cdh/resource/default/kbsres1/key1
{{% /tab %}}

{{% tab header="Basic nginx" %}}

This example demonstrates how to verify if Helm chart is successfully starting the PodVM within the cloud provider.
It is the simplest example available for deployment.

Create an `nginx` deployment:

```bash
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

{{% /tab %}}

{{< /tabpane >}}

Ensure that the pod is up and running:

```bash
kubectl get pods -n default
```

You can verify that the PodVM was created by running the following command:

```bash
gcloud compute instances list
```

Here you should see the VM associated with the pod used by the example above.

## Uninstall

To uninstall Confidential Containers from GKE cluster, use the following commands:

1. Remove all pods with `kata-runtime` runtime class:

    ```bash
    kubectl get pods -A -o custom-columns='NAME:.metadata.name,NAMESPACE:.metadata.namespace,RUNTIMECLASS:.spec.runtimeClassName' \
    | grep kata-remote \
    | awk '{print $1, $2}' \
    | xargs -n 2 sh -c 'kubectl delete pod -n "$2" "$1"' _
    ```

2. Verify that all peer pod VMs are deleted:

   Use the following command to list all the peer pod VMs (VMs having prefix `podvm`) and status.

   ```bash
   gcloud compute instances list \
     --filter="name~'podvm.*'" \
     --format="table(name,zone,status)"
   ```

3. List deployed Confidential Containers Helm chart:

   > **Note**: This command assumes that only one Helm release is deployed in the `confidential-containers-system` namespace. 
   > If there are multiple releases, you may need to adjust the command to select the correct one.

   ```bash
   export HELM_COCO_CHART_NAME=$(helm list \
                                  -n confidential-containers-system \
                                  --short)
   ```

4. Delete Confidential Containers related Helm chart:

   ```bash
   helm uninstall ${HELM_COCO_CHART_NAME} \
     --namespace confidential-containers-system
   ```

5. Delete secret with provider credentials `my-provider-creds`:

   ```bash
   kubectl delete secret my-provider-creds \
     -n confidential-containers-system
   ```

6. Delete Confidential Containers related namespace:

   ```bash
   kubectl delete namespace confidential-containers-system
   ```

7. Delete the GKE cluster by running the following command and confirming the deletion when prompted:

   ```bash
   gcloud container clusters delete "${GKE_CLUSTER_NAME}" \
     --zone "${GCP_REGION}-a"
   ```

## Debug SSH connection

> **Note**: SSH connection is available **only** for debug image, which is built with enabled SSH server and added public key to `authorized_keys` file.
> If you want to have SSH access to the image, make sure to build debug image and upload it to Google using above instructions.

After creating debug image with enabled SSH, deploy CoCo with sample pod and use root account to access it:

> **Note**: **Remember** to add your public key to Google using CLI

1. Export environment variable `GCP_PODVM_IP` using below code:

   ```bash
   GCP_LATEST_PODVM=$(gcloud compute instances list \
                               --project="${GCP_PROJECT_ID}" \
                               --filter="name ~ ^podvm-" \
                               --sort-by=~creationTimestamp \
                               --limit=1 \
                               --format="value(name)")
   
   export GCP_PODVM_IP=$(gcloud compute instances describe "${GCP_LATEST_PODVM}" \
                           --project="${GCP_PROJECT_ID}" \
                           --zone="$(gcloud compute instances list \
                                       --project="${GCP_PROJECT_ID}" \
                                       --filter="name=${GCP_LATEST_PODVM}" \
                                       --format="value(zone)")" \
                           --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
   ```

2. Connect to debug image using SSH:

   ```bash
   ssh -i ./gcp_ssh_debug root@"$GCP_PODVM_IP"
   ```

## Troubleshooting

> **Note**: If your case is not covered in section below check the troubleshooting guide [here](../troubleshooting/).

### VM Doesn't Start

Starting with GKE version **1.27**, GCP configures containerd with the `discard_unpacked_layers=true` flag to optimize disk
usage by removing compressed image layers after they are unpacked. However, this can cause issues with PeerPods,
as the workload may fail to locate required layers.
To avoid this, disable the `discard_unpacked_layers` setting in the containerd configuration.

Most of the time you will see a generic message such as the following:
```text
Error: failed to create containerd container: error unpacking image: failed to extract layer sha256:<SHA>: failed to get reader from content store: content digest sha256:<SHA>: not found
```

To disable the `discard_unpacked_layers` setting in the `containerd` configuration on **Google Kubernetes Engine (GKE) version 1.27 or later**, follow these steps:

1. SSH to worker node [Google console](https://console.cloud.google.com/compute/instances)

2. Run command which will change the `discard_unpacked_layers` property to `false` in the containerd configuration file:

   ```bash
   sudo sed -i 's/discard_unpacked_layers = true/discard_unpacked_layers = false/' /etc/containerd/config.toml
   ```

3. Verify changed property:

    ```bash
    sudo cat /etc/containerd/config.toml | grep discard_unpacked_layers
    ```

4. Restart containerd using below command:

    ```bash
    sudo systemctl restart containerd
    ```
