---
title: GCP
description: 在 GCP 上使用 Cloud API Adaptor (CAA) 的 Peer Pods Helm Chart
categories:
  - examples
tags:
  - helm
  - caa
  - gcp
  - gke
---

> **说明：** 本文为英文文档的中文译版，英文原版请参见 [GCP 示例（英文版）](https://confidentialcontainers.org/docs/examples/gcp-simple/)。

本文将介绍如何在 Google Kubernetes Engine (GKE) 上配置 CAA（即 Peer Pods），具体包括：

- 一个基于 GKE 的单工作节点 Kubernetes 集群
- 运行在该 Kubernetes 集群上的 CAA
- 一个由 CAA PodVM 支持的示例应用

## 前提条件

安装所需工具：

- 安装 [kubectl](https://kubernetes.io/docs/tasks/tools/)
- 安装 [Helm](https://helm.sh/docs/intro/install)
- 安装 `gcloud` CLI [工具](https://cloud.google.com/sdk/docs/install)

Google Cloud 项目：

- 确保你已经创建了一个 Google Cloud 项目
- 记录项目 ID（导出为 `GCP_PROJECT_ID`）

## GCP 准备工作

先完成 Google 认证，并选择要使用的项目：

```bash
export GCP_PROJECT_ID="YOUR_PROJECT_ID"
gcloud auth login
gcloud config set project ${GCP_PROJECT_ID}
```

启用所需 API：

```bash
gcloud services enable container.googleapis.com --project=${GCP_PROJECT_ID}
```

创建一个具备所需权限的服务账号：

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

生成并保存凭证文件：

```bash
gcloud iam service-accounts keys create \
  ~/.config/gcloud/peerpods_application_key.json \
  --iam-account=peerpods@${GCP_PROJECT_ID}.iam.gserviceaccount.com
```

```bash
export GOOGLE_APPLICATION_CREDENTIALS=~/.config/gcloud/peerpods_application_key.json
```

配置后续要使用的其他环境变量。

设置区域：

```bash
export GCP_REGION="us-central1"
```

{{% alert title="Note" color="primary" %}}
之所以选择 `us-central1`，是因为它支持机密虚拟机。支持区域的完整列表请访问：
https://cloud.google.com/confidential-computing/confidential-vm/docs/supported-configurations#supported-zones
{{% /alert %}}

设置 PodVM 实例类型：

{{< tabpane text=true right=true persist=header >}}

{{% tab header="AMD SEV-SNP" %}}
```bash
export PODVM_INSTANCE_TYPE="n2d-standard-4"
export DISABLECVM=false
export GCP_CONFIDENTIAL_TYPE="SEV" # SEV or SEV_SNP
export GCP_DISK_TYPE="pd-standard"
```
{{% /tab %}}

{{% tab header="Intel TDX" %}}
```bash
export PODVM_INSTANCE_TYPE="c3-standard-4"
export DISABLECVM=false
export GCP_CONFIDENTIAL_TYPE="TDX"
export GCP_DISK_TYPE="pd-balanced"
```
{{% /tab %}}

{{% tab header="非机密计算" %}}
```bash
export PODVM_INSTANCE_TYPE="e2-medium"
export DISABLECVM=true
export GCP_CONFIDENTIAL_TYPE=""
export GCP_DISK_TYPE="pd-standard"
```
{{% /tab %}}

{{< /tabpane >}}

## 使用 GKE 部署 Kubernetes

使用 GKE 部署一个单节点 Kubernetes 集群：

```bash
gcloud container clusters create my-cluster \
  --zone ${GCP_REGION}-a \
  --machine-type "e2-standard-4" \
  --image-type UBUNTU_CONTAINERD \
  --num-nodes 1
```

为工作节点添加标签：

```bash
kubectl get nodes --selector='!node-role.kubernetes.io/master' -o name | \
xargs -I{} kubectl label {} node.kubernetes.io/worker=
```

{{% alert title="Note" color="primary" %}}
从 GKE 1.27 开始，GCP 会为 containerd 配置 `discard_unpacked_layers=true`，以节省磁盘空间（移除解包后的压缩镜像层）。但这可能会导致 PeerPods 出现问题，因为工作负载可能找不到所需镜像层。

为避免这个问题，请在 containerd 配置中禁用 `discard_unpacked_layers`。

如果遇到虚拟机未正常运行的问题，请查看本页的[故障排查](#故障排查)部分。
{{% /alert %}}

### 配置 VPC 网络

我们需要确保默认 VPC 网络中已经放通 15150 端口：

```bash
gcloud compute firewall-rules create allow-port-15150 \
    --project=${GCP_PROJECT_ID} \
    --network=default \
    --allow=tcp:15150
```

在生产场景中，建议限制来源 IP 范围，以降低安全风险。例如，可以把来源范围限制为特定 IP 地址或 CIDR 段：

```bash
gcloud compute firewall-rules create allow-port-15150-restricted \
   --project=${GCP_PROJECT_ID} \
   --network=default \
   --allow=tcp:15150 \
   --source-ranges=[YOUR_EXTERNAL_IP]
```

## 部署 CAA Helm Chart

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
此方式假定你已在本地准备好代码，请在终端中切换到 Cloud API Adaptor 的代码目录。
{{% /tab %}}

{{< /tabpane >}}

### 导出 PodVM 镜像版本

导出 peer pods 所用的 PodVM 镜像 ID。该变量告诉部署工具在 Google Cloud 中创建 peer pod 虚拟机时应使用哪个 PodVM 镜像版本。

镜像来自 CoCo 社区镜像库（或由你手动构建），并且必须与当前 CAA 发布版本匹配。

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**版本**：" disabled=true /%}}

{{% tab header="最新发布" %}}

导出以下环境变量，作为 PodVM 使用的镜像：

```bash
export PODVM_IMAGE_ID="/projects/it-cloud-gcp-prod-osc-devel/global/images/fedora-mkosi-tee-amd-1-11-0"
```

{{% /tab %}}

{{% tab header="最新构建" %}}

最新构建没有预构建的 PodVM 镜像。你需要按[说明](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/gcp#build-pod-vm-image)构建 PodVM 镜像。镜像构建完成后，将镜像 ID 导出到环境变量 `PODVM_IMAGE_ID`。

{{% /tab %}}

{{% tab header="DIY" %}}

如果你修改了会影响 PodVM 镜像的 CAA 代码，并希望部署这些改动，请按[说明](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/gcp#build-pod-vm-image)构建 PodVM 镜像。镜像构建完成后，将镜像 ID 导出到环境变量 `PODVM_IMAGE_ID`。

{{% /tab %}}

{{< /tabpane >}}

#### 导出 CAA 容器镜像路径

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

如果你修改了 CAA 代码并希望部署这些改动，请按[这些说明](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/README.md#building-custom-cloud-api-adaptor-image)构建容器镜像。镜像构建完成后，导出环境变量 `CAA_IMAGE` 和 `CAA_TAG`。

{{% /tab %}}

{{< /tabpane >}}

### 填充 `providers/gcp.yaml` 文件

全部可用配置项可在以下两个位置找到：

- [主 chart values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/values.yaml)
- [GCP 专属 values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/gcp.yaml)

运行以下命令更新 [`providers/gcp.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/gcp.yaml) 文件：

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

   所需 key 请参见 [providers/gcp-secrets.yaml.template](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/gcp-secrets.yaml.template)。

   ```bash
   kubectl create secret generic my-provider-creds \
    -n confidential-containers-system \
    --from-file=GCP_CREDENTIALS="${GOOGLE_APPLICATION_CREDENTIALS}"
   ```

3. 安装 Helm Chart：

   下面命令使用了 `-f` 和 `--set` 这两个自定义选项，其含义可参考[这里](../../getting-started/installation/advanced_configuration)。

   ```bash
   helm install peerpods . \
     -f providers/gcp.yaml \
     --set secrets.mode=reference \
     --set secrets.existingSecretName=my-provider-creds \
     --dependency-update \
     -n confidential-containers-system
   ```

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

{{< tabpane text=true right=true persist=header >}}

{{% tab header="CoCo 机密获取"  %}}
本示例展示了一个更完整的部署方式：结合 TEE、机密虚拟机以及 `kata-remote` RuntimeClass，演示如何部署示例 Pod 并在机密计算环境中安全获取机密。

### 准备 init data 配置

Peer Pods 现已支持 init data。你可以通过注解 `io.katacontainers.config.hypervisor.cc_init_data` 传入所需配置文件（`aa.toml`、`cdh.toml` 和 `policy.rego`）。下面给出配置和使用示例。

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

请确认策略正确，并且 KBC URL 已指向你的 Key Broker Service。

然后对 `initdata.toml` 进行编码并保存为变量：

```bash
INITDATA=$(cat initdata.toml | gzip | base64 -w0)
```

使用以下命令部署 Pod：

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

### 从 Trustee 获取机密

Pod 成功携带 `initdata` 部署后，你可以在 Pod 内部从 Trustee 服务获取机密。使用以下命令获取指定机密：

```bash
kubectl exec -it example-pod -- curl http://127.0.0.1:8006/cdh/resource/default/kbsres1/key1
```
{{% /tab %}}

{{% tab header="基础 nginx" %}}

本示例是最基础的部署验证，用于确认 Helm Chart 是否已在云厂商侧成功启动 PodVM。

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

{{% /tab %}}

{{< /tabpane >}}

确认 pod 已成功启动：

```bash
kubectl get pods -n default
```

你可以通过运行以下命令确认 PodVM 是否已经创建：

```bash
gcloud compute instances list
```

此时你应该能看到与上述示例 Pod 对应的虚拟机。

## 清理

删除所有使用 `kata-remote` RuntimeClass 运行的 Pod。可以使用以下命令：

```bash
kubectl get pods -A -o custom-columns='NAME:.metadata.name,NAMESPACE:.metadata.namespace,RUNTIMECLASS:.spec.runtimeClassName' | grep kata-remote | awk '{print $1, $2}'
```

确认所有 peer-pod VM 都已删除。你可以使用以下命令列出所有 peer-pod VM（名称前缀为 `podvm`）及其状态：

```bash
gcloud compute instances list \
  --filter="name~'podvm.*'" \
  --format="table(name,zone,status)"
```

运行以下命令删除 GKE 集群：

```bash
gcloud container clusters delete my-cluster --zone ${GCP_REGION}-a
```

## 故障排查

> **说明：** 如果你的问题不在下面的范围内，请查看[这里](../troubleshooting/)的故障排查指南。

### 虚拟机未启动

从 GKE **1.27** 开始，GCP 会为 containerd 配置 `discard_unpacked_layers=true`，以节省磁盘空间（移除解包后的压缩镜像层）。但这可能会导致 PeerPods 出现问题，因为工作负载可能找不到所需镜像层。
为避免这个问题，请在 containerd 配置中禁用 `discard_unpacked_layers`。

大多数情况下，你会看到类似下面的通用报错：

```text
Error: failed to create containerd container: error unpacking image: failed to extract layer sha256:<SHA>: failed to get reader from content store: content digest sha256:<SHA>: not found
```

要在 **Google Kubernetes Engine (GKE) 1.27 及以上版本**中禁用 `containerd` 配置里的 `discard_unpacked_layers`，请按以下步骤操作：

1. 在 [Google Console](https://console.cloud.google.com/compute/instances) 中 SSH 登录工作节点
2. 运行命令 `sudo sed -i 's/discard_unpacked_layers = true/discard_unpacked_layers = false/' /etc/containerd/config.toml`
3. 运行 `sudo cat /etc/containerd/config.toml | grep discard_unpacked_layers` 验证修改结果
4. 重启 containerd：`sudo systemctl restart containerd`
