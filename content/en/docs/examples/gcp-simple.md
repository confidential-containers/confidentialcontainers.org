---
title: GCP
description: Cloud API Adaptor (CAA) on GCP
categories:
- examples
tags:
- caa
- gcp
- gke
---

This documentation will walk you through setting up CAA (a.k.a. Peer Pods) on
Google Kubernetes Engine (GKE). It explains how to deploy:

- A single worker node Kubernetes cluster using GKE 
- CAA on that Kubernetes cluster
- A sample application backed by a CAA pod VM

## Pre-requisites

1. **Install Required Tools**:
   - [Google Cloud CLI (`gcloud`)](https://cloud.google.com/sdk/docs/install)
   - [kubectl](https://kubernetes.io/docs/tasks/tools/)
2. **Google Cloud Project**:
   - Ensure you have a Google Cloud project created.
   - Note the Project ID (export it as `GCP_PROJECT_ID`).

## GCP Preparation

Start by authenticating with Google and choosing your project:

```bash
export GCP_PROJECT_ID="YOUR_PROJECT_ID"
gcloud auth login
gcloud config set project ${GCP_PROJECT_ID}
```

Enable the necessary API:

```bash
gcloud services enable container.googleapis.com --project=${GCP_PROJECT_ID}
```

Create a service account with the necessary permissions:

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

Generate and save the credentials file:

```bash
gcloud iam service-accounts keys create \
  ~/.config/gcloud/peerpods_application_key.json \
  --iam-account=peerpods@${GCP_PROJECT_ID}.iam.gserviceaccount.com
```

```bash
export GOOGLE_APPLICATION_CREDENTIALS=~/.config/gcloud/peerpods_application_key.json
```

Configure additional environment variables that will be used later.

Set the region:

```bash
export GCP_REGION="us-central1"
```
 
{{% alert title="Note" color="primary" %}}
"us-central1" was chosen because supports Confidential VMs. For a
complete list of supported regions visit
https://cloud.google.com/confidential-computing/confidential-vm/docs/supported-configurations#supported-zones
{{% /alert %}}

Set the PodVM instance type:

{{< tabpane text=true right=true persist=header >}}

{{% tab header="AMD SEV-SNP" %}}
```bash
export PODVM_INSTANCE_TYPE="n2d-standard-4"
export DISABLECVM=false
```
{{% /tab %}}

{{% tab header="Non-Confidential" %}}
```bash
export PODVM_INSTANCE_TYPE="e2-medium"
export DISABLECVM=true
```
{{% /tab %}}

{{< /tabpane >}}

## Deploy Kubernetes Using GKE

Deploy a single node Kubernetes cluster using GKE:

```bash
gcloud container clusters create my-cluster \
  --zone ${GCP_REGION}-a \
  --machine-type "e2-standard-4" \
  --image-type UBUNTU_CONTAINERD \
  --num-nodes 1
```

Label the worker nodes:

```bash
kubectl get nodes --selector='!node-role.kubernetes.io/master' -o name | \
xargs -I{} kubectl label {} node.kubernetes.io/worker=
```

{{% alert title="Note" color="primary" %}}
Starting with GKE version 1.27, GCP configures containerd with the
`discard_unpacked_layers=true` flag to optimize disk usage by removing
compressed image layers after they are unpacked. However, this can cause
issues with PeerPods, as the workload may fail to locate required layers. To
avoid this, disable the `discard_unpacked_layers` setting in the containerd
configuration.
{{% /alert %}}

### Configure VPC network

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
   --project=${PROJECT_ID} \
   --network=default \
   --allow=tcp:15150 \
   --source-ranges=[YOUR_EXTERNAL_IP]
```

## Deploy the CoCo Operator with PeerPods Runtime

Deploy the CoCo operator. Usually it’s the same version as CAA, but it can be
adjusted.

```bash
export CAA_VERSION="0.11.0"
export COCO_OPERATOR_VERSION="${CAA_VERSION}"

kubectl apply -k "github.com/confidential-containers/operator/config/release?ref=v${COCO_OPERATOR_VERSION}"
kubectl apply -k "github.com/confidential-containers/operator/config/samples/ccruntime/peer-pods?ref=v${COCO_OPERATOR_VERSION}"
```

## Deploy the Cloud API Adaptor (CAA)

### Download the CAA Deployment Artifacts

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

```bash
export CAA_VERSION="0.11.0"
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


### Configure the CAA PodVM image

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

Export this environment variable to use for the PodVM:

```bash
export PODVM_IMAGE_ID="/projects/it-cloud-gcp-prod-osc-devel/global/images/fedora-mkosi-tee-amd-1-11-0"
```

{{% /tab %}}

{{% tab header="Latest Build" %}}

There are no pre-built PodVM image for latest builds. You'll need to follow
[these
instructions](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/gcp#build-pod-vm-image)
to build the PodVM image. Once image build is finished then export image id to
the environment variable `PODVM_IMAGE_ID`.

{{% /tab %}}

{{% tab header="DIY" %}}

If you have made changes to the CAA code that affects the pod VM image and you
want to deploy those changes then follow [these
instructions](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/gcp#build-pod-vm-image)
to build the PodVM image. Once image build is finished then export image id to
the environment variable `PODVM_IMAGE_ID`.

{{% /tab %}}

{{< /tabpane >}}

#### Configure the CAA container image

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

Export the following environment variable to use the image built by the CAA CI
on each merge to main:

```bash
export CAA_IMAGE="quay.io/confidential-containers/cloud-api-adaptor"
```

Find an appropriate tag of pre-built image suitable to your needs [here](https://quay.io/repository/confidential-containers/cloud-api-adaptor?tab=tags&tag=latest).

```bash
export CAA_TAG=""
```

> **Caution**: You can also use the `latest` tag but it is **not** recommended,
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

### Create the GCP credentials file

Copy the Application Credentials to the GCP overlay folder:

```bash
cp $GOOGLE_APPLICATION_CREDENTIALS install/overlays/gcp/GCP_CREDENTIALS
```

### Populate the `kustomization.yaml` file

Run the following command to update the
[`kustomization.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/overlays/gcp/kustomization.yaml)
file:

```bash
cat <<EOF > install/overlays/gcp/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
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
  - CLOUD_PROVIDER="gcp"
  - PODVM_IMAGE_NAME="${PODVM_IMAGE_ID}"
  - GCP_PROJECT_ID="${GCP_PROJECT_ID}"
  - GCP_ZONE="${GCP_REGION}-a"
  - GCP_MACHINE_TYPE="${PODVM_INSTANCE_TYPE}"
  - DISABLECVM="${DISABLECVM}"
  - GCP_NETWORK="global/networks/default"
secretGenerator:
- name: peer-pods-secret
  namespace: confidential-containers-system
  files:
  - GCP_CREDENTIALS
EOF
```

### Deploy CAA on the Kubernetes cluster

Run the following command to deploy CAA:

```bash
kubectl apply -k "install/overlays/gcp"
```

Verify the deployment:

```bash
kubectl get pods -n confidential-containers-system
```

Verify that the `runtimeclass` is created after deploying CAA:

```bash
kubectl get runtimeclass
```

Once you can find a `runtimeclass` named `kata-remote` then you can be sure
that the deployment was successful. A successful deployment will look like
this:

```console
$ kubectl get runtimeclass
NAME          HANDLER       AGE
kata-remote   kata-remote   7m18s
```

Generic CAA deployment instructions are also described
[here](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/install).

## Run a sample application

{{< tabpane text=true right=true persist=header >}}

{{% tab header="CoCo Secret Retrieval"  %}}
This example showcases a more advanced deployment using TEE and confidential
VMs with the kata-remote runtime class. It demonstrates how to deploy a sample
pod and retrieve a secret securely within a confidential computing environment.

### Prepare the init data configuration

PeerPods now supports init data, you can pass the required configuration files
(`aa.toml`, `cdh.toml`, and `policy.rego`) via the
`io.katacontainers.config.runtime.cc_init_data` annotation. Below is an example
of the configuration and usage.

```bash
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

Make sure you have the right policy and KBC URL is pointing to your Key Broker
Service.

Now, encode the `initdata.toml` and store it in a variable

```bash
INITDATA=$(base64 -w0 initdata.toml)
```

Deploy the pod with:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  annotations:
    io.katacontainers.config.runtime.cc_init_data: "$INITDATA"
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
### Fetching Secrets from Trustee

Once the pod is successfully deployed with the `initdata`, you can retrieve
secrets from the Trustee service running inside the pod. Use the following
command to fetch a specific secret:

```bash
kubectl exec -it example-pod -- curl http://127.0.0.1:8006/cdh/resource/default/kbsres1/key1
{{% /tab %}}

{{% tab header="Basic nginx"  %}}

This example demonstrates how to verify if CAA is successfully starting the
PodVM within the cloud provider. It is the simplest example available for
deployment.

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
{{% /tab %}}

{{< /tabpane >}}

Ensure that the pod is up and running:

```bash
kubectl get pods -n default
```

You can verify that the PodVM was created by running the following
command:

```bash
gcloud compute instances list
```

Here you should see the VM associated with the pod used by the example above.

> **Note**: If you run into problems then check the troubleshooting guide
> [here](../troubleshooting/).

## Cleanup

Delete all running pods using the runtimeclass `kata-remote`. You can use the
following command for the same:

```bash
kubectl get pods -A -o custom-columns='NAME:.metadata.name,NAMESPACE:.metadata.namespace,RUNTIMECLASS:.spec.runtimeClassName' | grep kata-remote | awk '{print $1, $2}'
```

Verify that all peer-pod VMs are deleted. You can use the following command to
list all the peer-pod VMs (VMs having prefix `podvm`) and status:

```bash
gcloud compute instances list \
  --filter="name~'podvm.*'" \
  --format="table(name,zone,status)"
```

Delete the GKE cluster by running the following command:

```bash
gcloud container clusters delete my-cluster --zone ${GCP_REGION}-a
```
