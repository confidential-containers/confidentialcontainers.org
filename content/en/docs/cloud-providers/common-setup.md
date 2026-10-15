---
title: Common Cloud API Adaptor setup
linkTitle: Common setup
description: Setup steps shared by all Cloud API Adaptor cloud-provider deployments
categories:
  - cloud-providers
tags:
  - helm
  - caa
weight: 1
---

Use this page together with one provider guide. The provider guide tells you when to complete each shared step and supplies the cloud-specific cluster, network, identity, PodVM image, and Helm values.

## Prerequisites

Install the tools used by every provider guide:

- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- `curl`, `git`, and `jq`

Install the additional tools for your provider:

| Provider | Required tools |
| --- | --- |
| [AWS]({{< relref "aws-simple.md" >}}) | [`aws`](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and [`eksctl`](https://eksctl.io/installation/) |
| [Azure]({{< relref "azure-simple.md" >}}) | [`az`](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) and `sipcalc` |
| [Google Cloud]({{< relref "gcp-simple.md" >}}) | [`gcloud`](https://cloud.google.com/sdk/docs/install) |
| [Alibaba Cloud]({{< relref "alibaba-cloud-simple.md" >}}) | [`aliyun`](https://www.alibabacloud.com/help/en/cli/installation-guide/) |

Authenticate the provider CLI before creating cloud resources. Continue with your provider guide to prepare the cluster, networking, and identity.

## Prepare the CAA deployment

Complete the following steps when the provider guide directs you to prepare the shared deployment files. 

{{% alert title="Important" color="primary" %}}
Keep the same terminal session: later provider-specific commands use the variables exported here.
{{% /alert %}}

### Download the Helm deployment artifacts

Choose one source. Each option leaves the terminal in the `peerpods` Helm chart directory.

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

Use the latest stable release:

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

Use the current `main` branch for development or testing:

```bash
export CAA_BRANCH="main"
curl -LO "https://github.com/confidential-containers/cloud-api-adaptor/archive/refs/heads/${CAA_BRANCH}.tar.gz"
tar -xvzf "${CAA_BRANCH}.tar.gz"
cd "cloud-api-adaptor-${CAA_BRANCH}/src/cloud-api-adaptor/install/charts/peerpods"
```

{{% /tab %}}

{{% tab header="Local Checkout" %}}

Use an existing checkout when testing your own changes:

```bash
cd <cloud-api-adaptor-checkout>/src/cloud-api-adaptor/install/charts/peerpods
```

{{% /tab %}}
{{< /tabpane >}}

### Select the CAA container image

Use an image that matches the source selected above.

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

```bash
export CAA_IMAGE="quay.io/confidential-containers/cloud-api-adaptor"
export CAA_TAG="v${CAA_VERSION}-amd64"
```

{{% /tab %}}

{{% tab header="Latest Build" %}}

Select an architecture-appropriate CI tag from the [CAA image repository](https://quay.io/repository/confidential-containers/cloud-api-adaptor?tab=tags&tag=latest), then export it:

```bash
export CAA_IMAGE="quay.io/confidential-containers/cloud-api-adaptor"
export CAA_TAG="<CI_IMAGE_TAG>"
```

> **Caution**: You can also use the `latest` tag, but it is **not** recommended,
> because of its lack of version control and potential for unpredictable
> updates, impacting stability and reproducibility in deployments.

{{% /tab %}}

{{% tab header="Custom Image" %}}

Follow the [custom CAA image instructions](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/README.md#building-custom-cloud-api-adaptor-image), then export your image location:

```bash
export CAA_IMAGE="<REGISTRY>/<IMAGE>"
export CAA_TAG="<TAG>"
```

{{% /tab %}}
{{< /tabpane >}}

### Install cert-manager

> **Note:** Skip this step if cert-manager is already installed in the cluster.

The Peer Pods chart uses cert-manager for webhook certificates.

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
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=cert-manager \
  -n cert-manager \ 
  --timeout=300s
```

### Prepare the installation namespace

Create the namespace and add the metadata Helm needs to manage it:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: confidential-containers-system
  labels:
    app.kubernetes.io/managed-by: Helm
  annotations:
    meta.helm.sh/release-name: peerpods
    meta.helm.sh/release-namespace: confidential-containers-system
EOF
```

Return to your provider guide to configure the PodVM image, machine type, credentials, and provider values before installing the chart.

