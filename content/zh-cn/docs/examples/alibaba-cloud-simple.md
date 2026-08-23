---
title: 阿里云
description: 在阿里云上部署 Cloud API Adaptor (CAA)
categories:
- examples
tags:
- caa
- alibaba cloud
- ack
---

> **说明：** 本文为英文文档的中文译版，英文原版请参见[阿里云示例（英文版）](https://confidentialcontainers.org/docs/examples/alibaba-cloud-simple/)。

本文将介绍如何在阿里云容器服务 Kubernetes 版（ACK）和阿里云弹性计算服务（ECS）上部署 CAA（即 Peer Pods），具体包括：

- ACK 托管集群中的一个工作节点
- 运行在该 Kubernetes 集群上的 CAA
- 一个由运行在 ECS 上 CAA PodVM 支撑的 Nginx Pod

> **说明：** 请在目录 `src/cloud-api-adaptor` 下运行以下命令。

> **说明：** 当前机密计算实例已在[部分地域](https://www.alibabacloud.com/help/en/ecs/user-guide/build-a-tdx-confidential-computing-environment)提供。

> **说明：** 阿里云官方文档可参考[这里](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/implement-caa-confidential-container-solution-using-confidential-vms)。

## 前提条件

安装所需工具：

- 安装 `aliyun` CLI [工具](https://www.alibabacloud.com/help/en/cli/installation-guide/?spm=a2c63.p38356.help-menu-29991.d_2.28f346a6IMqkop)并[配置凭证](https://www.alibabacloud.com/help/en/cli/configure-credentials)
- 准备一个带 Bucket 的 `aliyun` OSS 存储

## 创建 PodVM 镜像

> **说明：**
> 在 `cn-hongkong` 地域中，版本 `0.22.0` 已提供一个预构建的社区镜像（id: `m-j6c3kfz2vhu6ze04wacb`），可直接用于测试。
> 也可以使用下方[导出 PodVM 镜像版本](#export-podvm-image-version)中列出的各地域镜像 ID。

如果你希望自行构建 PodVM 镜像，请按以下步骤操作。当前 PodVM 构建使用 [mkosi](https://github.com/systemd/mkosi)，目标系统为 **Ubuntu 26.04**（`resolute`）。前提条件和自定义选项请参见 [PodVM README](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/podvm/README.md)。

1. 构建 Ubuntu 26.04 PodVM 镜像。

   ```bash
   cd podvm
   make
   ```

   此命令将构建 PodVM 二进制文件和操作系统镜像，生成的 qcow2 文件路径为：

   ```text
   podvm/build/podvm-ubuntu-amd64.qcow2
   ```

   若二进制文件已存在，仅重新构建操作系统镜像：

   ```bash
   cd podvm
   make image
   ```

2. 上传到 OSS，并创建 ECS 镜像。

   在 `src/cloud-api-adaptor` 目录下，将 qcow2 文件上传到 OSS（对象存储服务）：

   ```bash
   cd ..
   export REGION_ID=<region-id>
   export IMAGE_FILE=podvm/build/podvm-ubuntu-amd64.qcow2
   export BUCKET=<OSS-bucket-name>
   export OBJECT=<object-name>

   aliyun oss cp ${IMAGE_FILE} oss://${BUCKET}/${OBJECT}
   ```

   然后将该镜像文件导入为 ECS 镜像：

   ```bash
   export IMAGE_NAME=$(basename ${IMAGE_FILE%.*})
   aliyun ecs ImportImage --ImageName ${IMAGE_NAME} \
       --region ${REGION_ID} --RegionId ${REGION_ID} \
       --BootMode UEFI \
       --DiskDeviceMapping.1.OSSBucket ${BUCKET} --DiskDeviceMapping.1.OSSObject ${OBJECT} \
       --Features.NvmeSupport supported \
       --method POST --force

   export POD_IMAGE_ID=<ImageId>
   ```

## 构建 CAA 开发镜像

如果你希望自行构建 CAA DaemonSet 镜像：

```bash
export registry=<registry-address>
export RELEASE_BUILD=true
export CLOUD_PROVIDER=alibabacloud
make image
```

请记录该镜像使用的 tag，后续会用到。

## 使用 ACK 托管集群部署 Kubernetes

1. 创建 ACK 托管集群。

   ```bash
   export CONTAINER_CIDR=172.18.0.0/16
   export REGION_ID=cn-beijing
   export ZONES='["cn-beijing-i"]'

   aliyun cs CreateCluster --header "Content-Type=application/json" --body "
   {
     \"cluster_type\":\"ManagedKubernetes\",
     \"name\":\"caa\",
     \"region_id\":\"${REGION_ID}\",
     \"zone_ids\":${ZONES},
     \"enable_rrsa\":true,
     \"container_cidr\":\"${CONTAINER_CIDR}\",
     \"addons\":[
       {
         \"name\":\"flannel\"
       }
     ]
   }"

   export CLUSTER_ID=<cluster-id>
   export SECURITY_GROUP_ID=$(aliyun cs DescribeClusterDetail --ClusterId ${CLUSTER_ID} | jq -r ".security_group_id")
   ```

   等待集群创建完成。获取该集群的 vSwitch ID，然后为集群添加一个工作节点。

2. 为集群所在 VPC 添加公网访问能力。

   ```bash
   export VPC_ID=$(aliyun cs DescribeClusterDetail --ClusterId ${CLUSTER_ID} | jq -r ".vpc_id")
   export VSWITCH_ID=$(echo ${VSWITCH_IDS} | sed 's/[][]//g' | sed 's/"//g')
   aliyun vpc CreateNatGateway \
     --region ${REGION_ID} \
     --RegionId ${REGION_ID} \
     --VpcId ${VPC_ID} \
     --NatType Enhanced \
     --VSwitchId ${VSWITCH_ID} \
     --NetworkType internet

   export GATEWAY_ID="<NatGatewayId>"
   export SNAT_TABLE_ID="<SnatTableId>"

   # 公网 IP 带宽（Mbps）
   export BAND_WIDTH=5
   aliyun vpc AllocateEipAddress \
     --region ${REGION_ID} \
     --RegionId ${REGION_ID} \
     --Bandwidth ${BAND_WIDTH}

   export EIP_ID="<AllocationId>"
   export EIP_ADDRESS="<EipAddress>"

   aliyun vpc AssociateEipAddress \
     --region ${REGION_ID} \
     --RegionId ${REGION_ID} \
     --AllocationId ${EIP_ID} \
     --InstanceId ${GATEWAY_ID} \
     --InstanceType Nat

   aliyun vpc CreateSnatEntry \
     --region ${REGION_ID} \
     --RegionId ${REGION_ID} \
     --SnatTableId ${SNAT_TABLE_ID} \
     --SourceVSwitchId ${VSWITCH_ID} \
     --SnatIp ${EIP_ADDRESS}
   ```

3. 授予角色权限。

   为集群工作节点授予相应角色权限，使其能够创建 ECS 实例。

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

导出 peer pods 所用的 PodVM 镜像 ID。该变量告诉部署工具在阿里云中创建 peer pod 虚拟机时应使用哪个 PodVM 镜像版本。

```bash
export IMAGEID="m-j6c3kfz2vhu6ze04wacb"
```

> **说明：** 阿里云会预先构建这些镜像，不同地域使用不同的镜像 ID。
>
> | region | IMAGEID |
> |---|---|
> | cn-beijing | m-2ze2vxvrxsbue3sf8b02 |
> | cn-hongkong | m-j6c3kfz2vhu6ze04wacb |
> | cn-hangzhou | m-bp146ws6x3iyuwjvbdw2 |
> | ap-southeast-1 | m-t4ng1w8ipua3c0o57hor |

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

```bash
export PODVM_INSTANCE_TYPE="ecs.g8i.xlarge"
export DISABLECVM="false"
```

> **说明：** 更多支持机密计算的实例规格，请参考[官方文档](https://help.aliyun.com/en/ecs/user-guide/build-a-tdx-confidential-computing-environment?spm=a2c4g.11186623.0.0.219e6187iTmYIJ#31e97c05ee64f)。

### 填充 `providers/alibabacloud.yaml` 文件

全部可用配置项可在以下两个位置找到：

- [主 chart values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/values.yaml)
- [阿里云专属 values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/alibabacloud.yaml)

运行以下命令更新 [`providers/alibabacloud.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/alibabacloud.yaml) 文件：

```bash
cat <<EOF > providers/alibabacloud.yaml
provider: alibabacloud
image:
  name: "${CAA_IMAGE}"
  tag: "${CAA_TAG}"
providerConfigs:
   alibabacloud:
      IMAGEID: "${IMAGEID}"
      REGION: "${REGION_ID}"
      SECURITY_GROUP_IDS: "${SECURITY_GROUP_ID}"
      VSWITCH_ID: "${VSWITCH_ID}"
      DISABLECVM: ${DISABLECVM}
alibabacloud:
  rrsa:
    enable: true
EOF
```

> **说明：** 如果你不使用 RRSA 进行认证，请将 yaml 中的 `alibabacloud.rrsa.enable` 改为 `false`。

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

   所需 key 请参见 [providers/alibabacloud-secrets.yaml.template](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/alibabacloud-secrets.yaml.template)。

   > **说明：** 以下示例假定你使用 RRSA 进行认证，因此不需要提供 `ALIBABACLOUD_ACCESS_KEY_ID` 和 `ALIBABACLOUD_ACCESS_KEY_SECRET`，而是提供 `ALIBABA_CLOUD_ROLE_ARN` 和 `ALIBABA_CLOUD_OIDC_PROVIDER_ARN`。

   ```bash
   kubectl create secret generic my-provider-creds \
   -n confidential-containers-system \
   --from-literal=ALIBABA_CLOUD_ROLE_ARN=${ALIBABA_CLOUD_ROLE_ARN} \
   --from-literal=ALIBABA_CLOUD_OIDC_PROVIDER_ARN=${ALIBABA_CLOUD_OIDC_PROVIDER_ARN} \
   --from-literal=ALIBABA_CLOUD_OIDC_TOKEN_FILE=/var/run/secrets/ack.alibabacloud.com/rrsa-tokens/token
   ```

3. 安装 Helm Chart：

   下面命令使用了 `-f` 和 `--set` 这两个自定义选项，其含义可参考[这里](../../getting-started/installation/advanced_configuration)。

   ```bash
   helm install peerpods . \
     -f providers/alibabacloud.yaml \
     --set secrets.mode=reference \
     --set secrets.existingSecretName=my-provider-creds \
     --dependency-update \
     -n confidential-containers-system
   ```

通用的 Peer Pods Helm Chart 部署说明也可参考[这里](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/install/charts/peerpods/README.md)。

## 运行示例应用

### 确认 runtimeclass 已创建

部署 CAA 后，请确认 `runtimeclass` 已创建：

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
echo '
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  runtimeClassName: kata-remote
  containers:
  - name: nginx
    image: registry.openanolis.cn/openanolis/nginx:1.14.1-8.6
' | kubectl apply -f -
```

确认 pod 已成功启动：

```bash
kubectl get pods -n default
```

你可以通过运行以下命令确认 peer-pod VM 是否已经创建：

```bash
aliyun ecs DescribeInstances --RegionId ${REGION_ID} --InstanceName 'podvm-*'
```

此时你应该能看到与 pod `nginx` 对应的虚拟机。
如果遇到问题，请查看[故障排查指南](../docs/troubleshooting/README.md)。

## 远程证明

TODO

## 清理

删除所有使用 `runtimeClass` `kata-remote` 运行的 Pod。

确认所有 peer-pod VM 都已删除。你可以使用以下命令列出所有 peer-pod VM（名称前缀为 `podvm`）及其状态：

```bash
aliyun ecs DescribeInstances --RegionId ${REGION_ID} --InstanceName 'podvm-*'
```

运行以下命令删除 ACK 集群：

```bash
aliyun cs DELETE /clusters/${CLUSTER_ID} --region ${REGION_ID} --keep_slb false --retain_all_resources false --header "Content-Type=application/json;" --body "{}"
```