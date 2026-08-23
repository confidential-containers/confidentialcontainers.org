---
title: AWS
description: 在 AWS 上使用 Cloud API Adaptor (CAA) 的 Peer Pods Helm Chart
categories:
- examples
tags:
- helm
- caa
- aws
- eks
- irsa
---

> **说明：** 本文为英文文档的中文译版，英文原版请参见 [AWS 示例（英文版）](https://confidentialcontainers.org/docs/examples/aws-simple/)。

本文将介绍如何在 AWS Elastic Kubernetes Service (EKS) 上配置 CAA（即 Peer Pods），具体包括：

- 一个基于 Elastic Kubernetes Service (EKS) 的单工作节点 Kubernetes 集群
- 运行在该 Kubernetes 集群上的 CAA
- 一个由 CAA PodVM 支撑的 Nginx Pod

## 前提条件

安装所需工具：

- 安装 [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- 安装 [Helm](https://helm.sh/docs/intro/install)
- 安装 `aws` CLI [工具](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- 安装 `eksctl` CLI [工具](https://eksctl.io/installation/)
- 确保已安装 `curl`、`git` 和 `jq`

## AWS 准备工作

- 为 AWS CLI 访问设置 `AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`（或 `AWS_PROFILE`）以及 `AWS_REGION`

> **说明：** 除了静态凭证外，也可以在 EKS 上使用 [IRSA（IAM Roles for Service Accounts）](#配置认证)。使用 IRSA 时，CAA Pod 通过 OIDC 完成认证，无需在 Kubernetes Secret 中保存静态 AWS 密钥。集群初始化步骤中仍然需要 `AWS_REGION` 和临时凭证。

- 设置区域：

{{< tabpane text=true right=true persist=header >}}

{{% tab header="AMD SEV-SNP" %}}

```bash
export AWS_REGION="us-east-2"
```

> **说明：** 选择 `us-east-2` 区域，是因为这里既提供 AMD SEV-SNP 实例，也提供可直接使用的预构建 PodVM 镜像。

{{% /tab %}}

{{% tab header="非机密计算" %}}

```bash
export AWS_REGION="us-east-2"
```

> **说明：** 选择 `us-east-2` 区域，是因为这里提供可直接使用的预构建 PodVM 镜像。

{{% /tab %}}
{{< /tabpane >}}

### 使用 EKS 部署 Kubernetes

按需修改以下环境变量：

```bash
export CLUSTER_NAME="caa-$(date '+%Y%m%b%d%H%M%S')"
export CLUSTER_NODE_TYPE="m5.xlarge"
export CLUSTER_NODE_FAMILY_TYPE="Ubuntu2204"
export SSH_KEY=~/.ssh/id_rsa.pub
```

以下示例使用默认的 AWS VPC-CNI 创建 EKS 集群：

```bash
eksctl create cluster --name "$CLUSTER_NAME" \
    --node-type "$CLUSTER_NODE_TYPE" \
    --node-ami-family "$CLUSTER_NODE_FAMILY_TYPE" \
    --nodes 1 \
    --nodes-min 0 \
    --nodes-max 2 \
    --node-private-networking \
    --kubeconfig "$CLUSTER_NAME"-kubeconfig
```

等待集群创建完成。

为集群节点添加 `node.kubernetes.io/worker=` 标签：

```bash
for NODE_NAME in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  kubectl label node $NODE_NAME node.kubernetes.io/worker=
done
```

### 放通所需网络端口

```bash
EKS_VPC_ID=$(aws eks describe-cluster --name "$CLUSTER_NAME" \
--query "cluster.resourcesVpcConfig.vpcId" \
--output text)
echo $EKS_VPC_ID

EKS_CLUSTER_SG=$(aws eks describe-cluster --name "$CLUSTER_NAME" \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" \
  --output text)
echo $EKS_CLUSTER_SG

EKS_VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids "$EKS_VPC_ID" \
--query 'Vpcs[0].CidrBlock' --output text)
echo $EKS_VPC_CIDR

# agent-protocol-forwarder 端口
aws ec2 authorize-security-group-ingress --group-id "$EKS_CLUSTER_SG" --protocol tcp --port 15150 --cidr "$EKS_VPC_CIDR"

# vxlan 端口
aws ec2 authorize-security-group-ingress --group-id "$EKS_CLUSTER_SG" --protocol tcp --port 9000 --cidr "$EKS_VPC_CIDR"
aws ec2 authorize-security-group-ingress --group-id "$EKS_CLUSTER_SG" --protocol udp --port 9000 --cidr "$EKS_VPC_CIDR"
```

> **说明：**
>
> - 端口 `15150` 是 CAA 连接 PodVM 内 `agent-protocol-forwarder` 时使用的默认端口。
> - 端口 `9000` 是 CAA 使用的 VXLAN 端口。请确保它与 Kubernetes CNI 使用的 VXLAN 端口不冲突。

### 配置认证

选择 CAA 与 AWS 交互时的认证方式。静态凭证方式通过 Kubernetes Secret 保存访问密钥；[IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 则通过 OIDC 联邦让 Pod 直接承担 IAM 角色，无需静态密钥。

{{< tabpane text=true right=true persist=header >}}

{{% tab header="静态凭证" %}}

确保环境中已设置 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`（或 `AWS_PROFILE`）。这些值会存入 Kubernetes Secret，供 CAA Pod 使用。

此处无需额外操作，Secret 将在后续的 [在 Kubernetes 集群中部署 Helm Chart](#在-kubernetes-集群中部署-helm-chart) 步骤中创建。

{{% /tab %}}

{{% tab header="IRSA (EKS)" %}}

IRSA 不再需要把长期 AWS 访问密钥保存在 Kubernetes Secret 中，是 EKS 上推荐使用的认证方式。

**启用 OIDC Provider**

检查 IAM OIDC provider 是否已经注册：

```bash
OIDC_ID=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --query "cluster.identity.oidc.issuer" \
  --output text | awk -F'/' '{print $NF}')

aws iam list-open-id-connect-providers | grep ${OIDC_ID}
```

如果命令没有返回结果，则创建 OIDC provider：

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --approve
```

导出账户 ID 和 OIDC provider，供后续步骤使用：

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

export OIDC_PROVIDER=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --query "cluster.identity.oidc.issuer" \
  --output text | sed 's|https://||')
```

**为 cloud-api-adaptor 创建 IAM Role**

```bash
export NAMESPACE="confidential-containers-system"
export CAA_SERVICE_ACCOUNT="cloud-api-adaptor"
export CAA_ROLE_NAME="CAA-IRSA-Role"
```

创建信任策略：

```bash
cat > /tmp/caa-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:${NAMESPACE}:${CAA_SERVICE_ACCOUNT}",
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF
```

创建 IAM 角色，并附加 `AmazonEC2FullAccess` 托管策略：

```bash
aws iam create-role \
  --role-name ${CAA_ROLE_NAME} \
  --assume-role-policy-document file:///tmp/caa-trust-policy.json \
  --description "IRSA role for Cloud API Adaptor on EKS"

aws iam attach-role-policy \
  --role-name ${CAA_ROLE_NAME} \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

> **说明：** `AmazonEC2FullAccess` 会授予较宽泛的 EC2 权限。用于生产环境时，强烈建议改用更小权限范围的自定义策略，仅授予 CAA 所需的 EC2 操作权限。更多配置方式请参考 [AWS IRSA 文档](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/docs/aws-irsa.md)。

导出 role ARN，供后续使用：

```bash
export CAA_ROLE_ARN=$(aws iam get-role \
  --role-name ${CAA_ROLE_NAME} \
  --query 'Role.Arn' \
  --output text)

echo "CAA Role ARN: ${CAA_ROLE_ARN}"
```

**为 Peerpod-ctrl 创建 IAM Role（可选）**

只有在部署 Peerpod-ctrl 时才需要这一步。

```bash
export CTRL_SERVICE_ACCOUNT="peerpodctrl-controller-manager"
export CTRL_ROLE_NAME="PeerpodCtrl-IRSA-Role"
```

创建信任策略：

```bash
cat > /tmp/peerpod-ctrl-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:${NAMESPACE}:${CTRL_SERVICE_ACCOUNT}",
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF
```

创建角色并附加权限：

```bash
aws iam create-role \
  --role-name ${CTRL_ROLE_NAME} \
  --assume-role-policy-document file:///tmp/peerpod-ctrl-trust-policy.json \
  --description "IRSA role for PeerPod Controller on EKS"

aws iam attach-role-policy \
  --role-name ${CTRL_ROLE_NAME} \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

> **说明：** `AmazonEC2FullAccess` 会授予较宽泛的 EC2 权限。用于生产环境时，强烈建议改用更小权限范围的自定义策略，仅授予 CAA 所需的 EC2 操作权限。更多配置方式请参考 [AWS IRSA 文档](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/docs/aws-irsa.md)。

导出 role ARN：

```bash
export CTRL_ROLE_ARN=$(aws iam get-role \
  --role-name ${CTRL_ROLE_NAME} \
  --query 'Role.Arn' \
  --output text)

echo "Controller Role ARN: ${CTRL_ROLE_ARN}"
```

{{% /tab %}}

{{< /tabpane >}}

## 部署 CAA Helm Chart

### 下载 CAA Helm 部署资源

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**版本**：" disabled=true /%}}

{{% tab header="最新发布" %}}

```bash
export CAA_VERSION="0.22.0"
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

导出 peer pods 所用的 PodVM 镜像 ID。该变量告诉部署工具在 AWS 中创建 peer pod 虚拟机时应使用哪个 PodVM 镜像版本。

镜像来自 CoCo 社区镜像库（或由你手动构建），并且必须与当前 CAA 发布版本匹配。

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**版本**：" disabled=true /%}}

{{% tab header="最新发布" %}}

在 `us-east-2` 区域中，我们提供了一个可用于 PoC 的预构建调试版 PodVM 镜像。可通过以下命令查询对应发布版本的 AMI ID：

```bash
export PODVM_AMI_ID=$(aws ec2 describe-images \
    --filters Name=name,Values="podvm-ubuntu-amd64-${CAA_VERSION//./-}" \
    --query 'Images[*].[ImageId]' --output text)

echo $PODVM_AMI_ID
```

{{% /tab %}}

{{% tab header="最新构建" %}}

最新构建没有预构建的 PodVM AMI。你需要自行构建 PodVM 镜像，然后按照[这里](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/podvm-mkosi)的说明创建 AMI。

为 AWS 构建 PodVM 镜像前，请记得设置 `TEE_PLATFORM=amd`。

镜像构建完成后，将镜像 ID 导出到环境变量 `PODVM_AMI_ID`。

{{% /tab %}}

{{% tab header="DIY" %}}

你可以按照[这里](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/podvm-mkosi)的说明构建自定义 PodVM 镜像。

为 AWS 构建 PodVM 镜像前，请记得设置 `TEE_PLATFORM=amd`。

镜像构建完成后，将镜像 ID 导出到环境变量 `PODVM_AMI_ID`。

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

如果你修改了 CAA 代码并希望部署这些改动，请按[这些说明](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/README.md#building-custom-cloud-api-adaptor-image)构建容器镜像。镜像构建完成后，导出环境变量 `CAA_IMAGE` 和 `CAA_TAG`。

{{% /tab %}}

{{< /tabpane >}}

### 选择 peer-pods 机型

{{< tabpane text=true right=true persist=header >}}
{{% tab header="AMD SEV-SNP" %}}

```bash
export PODVM_INSTANCE_TYPE="m6a.large"
export DISABLECVM="false"
```

更多 AMD SEV-SNP 机型可参考[这份](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snp-work.html) AWS 文档。

{{% /tab %}}

{{% tab header="非机密计算" %}}

```bash
export PODVM_INSTANCE_TYPE="t3.large"
export DISABLECVM="true"
```

{{% /tab %}}
{{< /tabpane >}}

### 填充 `providers/aws.yaml` 文件

全部可用配置项可在以下两个位置找到：

- [主 chart values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/values.yaml)
- [AWS 专属 values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/aws.yaml)

运行以下命令更新 [`providers/aws.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/aws.yaml) 文件：

```bash
cat <<EOF > providers/aws.yaml
provider: aws
image:
  name: "${CAA_IMAGE}"
  tag: "${CAA_TAG}"
providerConfigs:
   aws:
      DISABLECVM: ${DISABLECVM}
      PODVM_AMI_ID: "${PODVM_AMI_ID}"
      PODVM_INSTANCE_TYPE: "${PODVM_INSTANCE_TYPE}"
      VXLAN_PORT: 9000
EOF
```

### 安装 cert-manager

CAA 依赖 cert-manager。除非环境中已经安装，否则请使用以下方式部署：

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --wait \
  --timeout 5m
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

2. 创建凭证并安装 Helm Chart：

   下面命令使用了 `-f` 和 `--set` 这两个自定义选项，其含义可参考[这里](../../getting-started/installation/advanced_configuration)。

{{< tabpane text=true right=true persist=header >}}

{{% tab header="静态凭证" %}}

使用 `kubectl` 创建 Secret。所需 key 请参见 [providers/aws-secrets.yaml.template](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/aws-secrets.yaml.template)。

```bash
kubectl create secret generic my-provider-creds \
  -n confidential-containers-system \
  --from-literal=AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID} \
  --from-literal=AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY} \
  --from-file=id_rsa.pub=${SSH_KEY}
```

> **说明：** `--from-file=id_rsa.pub=${SSH_KEY}` 是可选项。它允许用户为排障目的通过 SSH 登录 PodVM。
> 该选项只对启用了调试功能的自定义 PodVM 镜像有效。预构建的 PodVM 镜像默认不启用 SSH 连接。

安装 Helm Chart：

```bash
helm install peerpods . \
  -f providers/aws.yaml \
  --set secrets.mode=reference \
  --set secrets.existingSecretName=my-provider-creds \
  --dependency-update \
  -n confidential-containers-system
```

{{% /tab %}}

{{% tab header="IRSA (EKS)" %}}

使用 IRSA 时，无需创建 AWS 凭证 Secret。CAA Pod 会通过[配置认证](#配置认证)章节中配置的 IAM 角色完成认证。

使用 IRSA 注解安装 Helm Chart：

```bash
helm install peerpods . \
  -f providers/aws.yaml \
  --set "daemonset.serviceAccount.annotations.eks\.amazonaws\.com/role-arn=${CAA_ROLE_ARN}" \
  --set "resourceCtrl.serviceAccount.annotations.eks\.amazonaws\.com/role-arn=${CTRL_ROLE_ARN}" \
  --dependency-update \
  -n confidential-containers-system
```

> **说明：** `resourceCtrl.serviceAccount.annotations` 这一行只有在部署 Peerpod-ctrl 时才需要。
> 如果不部署它，可以省略这个 `--set` 参数，并跳过前文中的 [为 Peerpod-ctrl 创建 IAM Role（可选）](#为-peerpod-ctrl-创建-iam-role可选) 步骤。

验证 IRSA 是否生效，可检查服务账号注解和 Pod 环境变量：

```bash
kubectl get serviceaccount cloud-api-adaptor \
  -n confidential-containers-system \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'
```

```bash
CAA_POD=$(kubectl get pods -n confidential-containers-system \
  -l app=cloud-api-adaptor \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n confidential-containers-system ${CAA_POD} -- env | grep AWS
```

输出中应包含 `AWS_WEB_IDENTITY_TOKEN_FILE` 和 `AWS_ROLE_ARN` 变量。

{{% /tab %}}

{{< /tabpane >}}

通用的 Peer Pods Helm Chart 部署说明也可参考[这里](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/install/charts/peerpods/README.md)。

## 运行示例应用

### 确认 RuntimeClass 已创建

部署 CAA 后，请确认已创建 `RuntimeClass`：

```bash
kubectl get runtimeclass
```

当你看到名为 `kata-remote` 的 `RuntimeClass` 时，就说明部署成功。成功输出类似如下：

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
aws ec2 describe-instances --filters "Name=tag:Name,Values=podvm*" \
   --query 'Reservations[*].Instances[*].[InstanceId, Tags[?Key==`Name`].Value | [0]]' --output table
```

此时你应该能看到与 pod `nginx` 对应的虚拟机。

> **说明：** 如果遇到问题，请查看[故障排查指南](../troubleshooting/)。

## 清理

删除所有使用 `kata-remote` `runtimeclass` 运行的 Pod。可以使用以下命令：

```bash
kubectl get pods -A -o custom-columns='NAME:.metadata.name,NAMESPACE:.metadata.namespace,RUNTIMECLASS:.spec.runtimeClassName' | grep kata-remote | awk '{print $1, $2}'
```

确认所有 peer-pod VM 都已删除。你可以使用以下命令列出所有 peer-pod VM（名称前缀为 `podvm`）及其状态：

```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=podvm*" \
--query 'Reservations[*].Instances[*].[InstanceId, Tags[?Key==`Name`].Value | [0], State.Name]' --output table
```

运行以下命令删除 EKS 集群：

```bash
eksctl delete cluster --name=$CLUSTER_NAME
```
