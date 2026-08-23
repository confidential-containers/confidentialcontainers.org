---
title: Azure
description: 在 Azure 上使用 Cloud API Adaptor (CAA) 的 Peer Pods Helm Chart
categories:
- examples
tags:
- helm
- caa
- azure
---

> **说明：** 本文为英文文档的中文译版，英文原版请参见 [Azure 示例（英文版）](https://confidentialcontainers.org/docs/examples/azure-simple/)。

本文将介绍如何在 Azure Kubernetes Service (AKS) 上配置 CAA（即 Peer Pods），具体包括：

- 一个基于 Azure Kubernetes Service (AKS) 的单工作节点 Kubernetes 集群
- 运行在该 Kubernetes 集群上的 CAA
- 一个由 CAA PodVM 支撑的 Nginx Pod

Confidential Containers 也支持将 Azure Key Vault 用作 Trustee 的资源后端。
[更多信息](../../attestation/resources/kbs-backed-by-akv)

## 前提条件

安装所需工具：

- 安装 [kubectl](https://kubernetes.io/docs/tasks/tools/)
- 安装 [Helm](https://helm.sh/docs/intro/install)
- 安装 `az` CLI [工具](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- 确保已安装 `curl`、`git`、`jq` 和 `sipcalc`

## Azure 准备工作

### 登录 Azure

以下许多步骤均需先登录 Azure 账号：

```bash
az login
```

获取你的订阅 ID：

```bash
export AZURE_SUBSCRIPTION_ID=$(az account show --query id --output tsv)
```

设置区域：

{{< tabpane text=true right=true persist=header >}}

{{% tab header="AMD SEV-SNP" %}}

```bash
export AZURE_REGION="eastus"
```

> **说明：** 选择 `eastus` 区域，是因为这里既提供 AMD SEV-SNP 实例，也提供可直接使用的预构建 PodVM 镜像。

{{% /tab %}}

{{% tab header="Intel TDX" %}}

```bash
export AZURE_REGION="eastus2"
```

> **说明：** 选择 `eastus2` 区域，是因为这里既提供 Intel TDX 实例，也提供可直接使用的预构建 PodVM 镜像。

{{% /tab %}}

{{% tab header="非机密计算" %}}

```bash
export AZURE_REGION="eastus"
```

> **说明：** 选择 `eastus` 区域，是因为这里提供可直接使用的预构建 PodVM 镜像。

{{% /tab %}}
{{< /tabpane >}}

### 资源组

> **说明：** 如果你已经有可用的资源组，可以跳过这一步。请将资源组名称导出到环境变量 `AZURE_RESOURCE_GROUP`。

运行以下命令创建 Azure 资源组：

```bash
export AZURE_RESOURCE_GROUP="caa-rg-$(date '+%Y%m%b%d%H%M%S')"

az group create \
  --name "${AZURE_RESOURCE_GROUP}" \
  --location "${AZURE_REGION}"
```

### 使用 AKS 部署 Kubernetes

按需修改以下环境变量：

```bash
export CLUSTER_NAME="caa-$(date '+%Y%m%b%d%H%M%S')"
export AKS_WORKER_USER_NAME="azuser"
export AKS_RG="${AZURE_RESOURCE_GROUP}-aks"
export SSH_KEY=~/.ssh/id_rsa.pub
```

> **说明：** 你也可以通过增加参数 `--vnet-subnet-id $MY_SUBNET_ID`，将工作节点部署到现有的 Azure Virtual Network (VNet) 和子网中。

将一个单工作节点的 AKS 集群部署到你刚刚创建的资源组中：

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

将 kubeconfig 下载到本地，以便用 `kubectl` 访问集群：

```bash
az aks get-credentials \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --name "${CLUSTER_NAME}"
```

### 用户分配身份与联合凭证

CAA 需要具备访问 Azure API 的权限。做法是将工作负载身份（workload identity）关联到 CAA 的服务账号上。该工作负载身份（即用户分配身份）将在下一步被授予创建虚拟机、获取镜像和访问网络的权限。

> **说明：** 如果你使用的是现有 AKS 集群，可能需要先配置工作负载身份（workload identity）和 OpenID Connect (OIDC)。可参考[指南](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster#update-an-existing-aks-cluster)。

先为 CAA 创建一个身份：

```bash
export AZURE_WORKLOAD_IDENTITY_NAME="${CLUSTER_NAME}-identity"

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

### 网络

承载 Pod 的虚拟机通常需要访问互联网服务，例如从公共 OCI 镜像仓库拉取镜像。你可以在 AKS 集群所在 VNet 中，为 AKS 子网旁边新建一个独立子网，再为该子网绑定带公网 IP 的 NAT 网关：

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

### AKS 资源组权限

为了让 CAA 能管理虚拟机，需为该身份授予虚拟机和网络相关权限，使其能够在 `$AZURE_RESOURCE_GROUP` 中创建虚拟机，并连接 `$AKS_RG` 中的 VNet。

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

使用 AKS 集群的 OIDC endpoint 为 CAA ServiceAccount 创建联合凭证：

```bash
export AKS_OIDC_ISSUER="$(az aks show \
  --name "${CLUSTER_NAME}" \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --query "oidcIssuerProfile.issuerUrl" \
  -otsv)"
```

```bash
az identity federated-credential create \
  --name "${CLUSTER_NAME}-federated" \
  --identity-name "${AZURE_WORKLOAD_IDENTITY_NAME}" \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --issuer "${AKS_OIDC_ISSUER}" \
  --subject system:serviceaccount:confidential-containers-system:cloud-api-adaptor \
  --audience api://AzureADTokenExchange
```

## 部署 CAA Helm Chart

> **说明：** 如果你的 Kubernetes 集群使用 Calico Container Network Interface (CNI)，请先按[说明](https://projectcalico.docs.tigera.io/networking/vxlan-ipip#configure-vxlan-encapsulation-for-all-inter-workload-traffic)为所有工作负载间流量配置 Virtual Extensible LAN (VXLAN) 封装。

### 下载 CAA Helm 部署资源

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**版本**：" disabled=true /%}}

{{% tab header="最新发布" %}}

```bash
export CAA_VERSION="0.17.0"
curl -LO "https://github.com/confidential-containers/cloud-api-adaptor/archive/refs/tags/v${CAA_VERSION}.tar.gz"
tar -xvzf "v${CAA_VERSION}.tar.gz"
cd "cloud-api-adaptor-${CAA_VERSION}/src/cloud-api-adaptor/install/charts/peerpods"
```

{{% /tab %}}

{{% tab header="最新构建" %}}

```bash
export CAA_BRANCH="main"
curl -LO "https://github.com/confidential-containers/cloud-api-adaptor/archive/refs/heads/${CAA_BRANCH}.tar.gz"
tar -xvzf "${CAA_BRANCH}.tar.gz"
cd "cloud-api-adaptor-${CAA_BRANCH}/src/cloud-api-adaptor/install/charts/peerpods"
```

{{% /tab %}}

{{% tab header="DIY" %}}
假如你已在本地准备好代码，请在终端中切换到 Cloud API Adaptor 的代码目录。
{{% /tab %}}

{{< /tabpane >}}

### 导出 PodVM 镜像版本

导出 peer pods 所用的 PodVM 镜像 ID。该变量告诉部署工具在 Azure 中创建 peer pod 虚拟机时应使用哪个 PodVM 镜像版本。

镜像来自 CoCo 社区镜像库（或由你手动构建），并且必须与当前 CAA 发布版本匹配。

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**版本**：" disabled=true /%}}

{{% tab header="最新发布" %}}

导出以下环境变量，作为 peer pod VM 使用的镜像：

```bash
export AZURE_IMAGE_ID="/CommunityGalleries/cococommunity-42d8482d-92cd-415b-b332-7648bd978eff/Images/peerpod-podvm-fedora/Versions/${CAA_VERSION}"
```

{{% /tab %}}

{{% tab header="最新构建" %}}

自动化任务会在每天 00:00 UTC 构建一次 PodVM 镜像。你可以导出以下环境变量来使用该镜像：

```bash
SUCCESS_TIME=$(curl -s \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/confidential-containers/cloud-api-adaptor/actions/workflows/azure-nightly-build.yml/runs?status=success" \
  | jq -r '.workflow_runs[0].updated_at')

export AZURE_IMAGE_ID="/CommunityGalleries/cocopodvm-d0e4f35f-5530-4b9c-8596-112487cdea85/Images/podvm_image0/Versions/$(date -u -jf "%Y-%m-%dT%H:%M:%SZ" "$SUCCESS_TIME" "+%Y.%m.%d" 2>/dev/null || date -d "$SUCCESS_TIME" +%Y.%m.%d)"
```

版本号格式为 `YYYY.MM.DD`，最新镜像对应今天或前一天的日期。

{{% /tab %}}

{{% tab header="DIY" %}}

如果你修改了会影响 PodVM 镜像的 CAA 代码，并希望部署这些改动，请按[这些说明](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/azure/build-image.md)构建 PodVM 镜像。
镜像构建完成后，将镜像 ID 导出到环境变量 `AZURE_IMAGE_ID`。

{{% /tab %}}

{{< /tabpane >}}

### 导出 CAA 容器镜像路径

定义要部署的 Cloud API Adaptor（CAA）容器镜像。
这些变量指定了部署工具所要拉取和运行的 CAA 镜像及其架构专属 tag。
tag 与 CAA 发布版本对应，以确保与所选 PodVM 镜像和配置兼容。

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**版本**：" disabled=true /%}}

{{% tab header="最新发布" %}}

导出以下环境变量以使用 CAA 最新发布镜像：

```bash
export CAA_IMAGE="quay.io/confidential-containers/cloud-api-adaptor"
export CAA_TAG="v${CAA_VERSION}-amd64"
```

{{% /tab %}}

{{% tab header="最新构建" %}}

导出以下环境变量，以使用每次合并到 `main` 后由 CAA CI 构建的镜像：

```bash
export CAA_IMAGE="quay.io/confidential-containers/cloud-api-adaptor"
```

你可以在[这里](https://quay.io/repository/confidential-containers/cloud-api-adaptor?tab=tags&tag=latest)找到适合需求的预构建镜像 tag。

```bash
export CAA_TAG=""
```

> **注意：** 你也可以使用 `latest` tag，但**不推荐**这样做，因为它缺少版本控制，可能引入不可预期的更新，影响部署稳定性和可复现性。

{{% /tab %}}

{{% tab header="DIY" %}}

如果你修改了 CAA 代码并希望部署这些改动，请按[说明](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/README.md#building-custom-cloud-api-adaptor-image)构建容器镜像。镜像构建完成后，导出环境变量 `CAA_IMAGE` 和 `CAA_TAG`。

{{% /tab %}}

{{< /tabpane >}}

### 选择 peer-pods 机型

{{< tabpane text=true right=true persist=header >}}
{{% tab header="AMD SEV-SNP" %}}

```bash
export AZURE_INSTANCE_SIZE="Standard_DC2as_v5"
export DISABLECVM="false"
```

更多 AMD SEV-SNP 机型可参考[Azure 文档](https://learn.microsoft.com/en-us/azure/virtual-machines/dasv5-dadsv5-series) 。

{{% /tab %}}

{{% tab header="Intel TDX" %}}

```bash
export AZURE_INSTANCE_SIZE="Standard_DC2es_v6"
export DISABLECVM="false"
```

更多 Intel TDX 机型可参考[Azure 文档](https://learn.microsoft.com/en-us/azure/virtual-machines/dcesv5-dcedsv5-series)。

{{% /tab %}}

{{% tab header="非机密计算" %}}

```bash
export AZURE_INSTANCE_SIZE="Standard_D2as_v5"
export DISABLECVM="true"
```

{{% /tab %}}
{{< /tabpane >}}

### 填充 `providers/azure.yaml` 文件

全部可用配置项可在以下两个位置找到：

- [主 chart values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/values.yaml)
- [Azure 专属 values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/azure.yaml)

运行以下命令更新 [`providers/azure.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/azure.yaml) 文件：

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
      AZURE_SUBSCRIPTION_ID: "${AZURE_SUBSCRIPTION_ID}"
      AZURE_INSTANCE_SIZE: "${AZURE_INSTANCE_SIZE}"
      DISABLECVM: ${DISABLECVM}
EOF
```

### 在 Kubernetes 集群中部署 Helm Chart

1. 创建由 Helm 管理的命名空间：

   ```bash
   kubectl apply -f - << EOF
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

2. 使用 `kubectl` 创建 Secret：

   所需 key 请参见 [providers/azure-secrets.yaml.template](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/azure-secrets.yaml.template)。

  > **说明：** 以下示例假定你使用工作负载身份（workload identity）进行认证，因此不需要提供 `AZURE_CLIENT_SECRET` 和 `AZURE_TENANT_ID`。

   ```bash
   kubectl create secret generic my-provider-creds \
    -n confidential-containers-system \
    --from-literal=AZURE_CLIENT_ID="${USER_ASSIGNED_CLIENT_ID}" \
    --from-file=id_rsa.pub=${SSH_KEY}
   ```

   > **说明：** `--from-file=id_rsa.pub=${SSH_KEY}` 是可选项。它允许用户为排障目的通过 SSH 登录 PodVM。
   > 该选项只对启用了调试功能的自定义 PodVM 镜像有效。预构建的 PodVM 镜像默认不启用 SSH 连接。

3. 安装 Helm Chart：

   下面命令使用了 `-f` 和 `--set` 这两个自定义选项，其含义可参考[这里](../../getting-started/installation/advanced_configuration)。

   ```bash
   helm install peerpods . \
     -f providers/azure.yaml \
     --set secrets.mode=reference \
     --set secrets.existingSecretName=my-provider-creds \
     --set-json daemonset.podLabels='{"azure.workload.identity/use":"true"}' \
     --dependency-update \
     -n confidential-containers-system
   ```

  > **说明：** 上述示例假定你使用工作负载身份（workload identity）进行认证。<br>
  > `--set-json daemonset.podLabels='{{"azure.workload.identity/use":"true"}}'` 参数**仅**在使用工作负载身份时才需要。

通用的 Peer Pods Helm Chart 部署说明也可参考[这里](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/install/charts/peerpods/README.md)。

## 运行示例应用

### 确认 RuntimeClass 已创建

部署 Peer Pods Helm Chart 后，请确认已创建 `runtimeclass`：

```bash
kubectl get runtimeclass
```

当你看到名为 `kata-remote` 的 `runtimeclass` 时，就说明部署成功。
成功输出类似如下：

```console
$ kubectl get runtimeclass
NAME          HANDLER       AGE
kata-remote   kata-remote   7m18s
```

### 部署工作负载

创建一个 `nginx` deployment：

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

确认 pod 已成功启动：

```bash
kubectl get pods -n default
```

你可以通过运行以下命令确认 peer pod VM 是否已经创建：

```bash
az vm list \
  --resource-group "${AZURE_RESOURCE_GROUP}" \
  --output table
```

此时你应该能看到与 pod `nginx` 对应的虚拟机。

> **说明：** 如果遇到问题，请查看[故障排查指南](../troubleshooting/)。

## PodVM 参考值

PodVM 镜像在构建过程中，会把预期的 PCR 测量值发布到 OCI 镜像仓库中。

### 前提条件

安装 [ORAS](https://oras.land/) 工具，以便从 OCI 镜像仓库拉取参考值；安装 [GitHub CLI](https://cli.github.com/)，以便验证构建来源。

### 验证构建来源

确认这些测量值是由官方仓库中可信的构建流程生成的。指定 `--format=json` 可以看到更详细的构建信息。

```bash
CAA_REPO="confidential-containers/cloud-api-adaptor"
OCI_REGISTRY="ghcr.io/${CAA_REPO}/measurements/azure/podvm:${CAA_VERSION}"
gh attestation verify -R "$CAA_REPO" "oci://${OCI_REGISTRY}"
```

### 获取参考值

这些 PCR 值可用于远程证明策略中，以校验 PodVM 镜像的完整性。

```bash
oras pull "$OCI_REGISTRY"
jq -r .measurements.sha256.pcr11 < measurements.json
0x58e8afdf5b105fc6b202eb8e537a9f1512a4b33cd5921171b518645a86ca5a75
```

## 清理

如果你希望清理整个环境，可以运行以下命令删除资源组：

```bash
az group delete \
  --name "${AZURE_RESOURCE_GROUP}" \
  --yes --no-wait
```
