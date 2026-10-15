---
title: Alibaba Cloud (ACK)
description: Cloud API Adaptor (CAA) on Alibaba Cloud
categories:
  - cloud-providers
tags:
  - caa
  - alibaba cloud
  - ack
---

This documentation will walk you through setting up CAA (a.k.a. Peer Pods) on Alibaba Cloud Container Service for Kubernetes (ACK) and Alibaba Cloud Elastic Compute Service (ECS). It explains how to deploy:

- One worker for ACK Managed Cluster
- CAA on that Kubernetes cluster
- An Nginx pod backed by CAA pod VM on ECS

> **Note:** Run the following commands from the following directory - `src/cloud-api-adaptor`

> **Note:** Now Confidential Computing instances are available [in some regions](https://www.alibabacloud.com/help/en/ecs/user-guide/build-a-tdx-confidential-computing-environment).

> **Note:** Official document from Alibaba Cloud can be found [here](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/implement-caa-confidential-container-solution-using-confidential-vms).

## Before you begin

Complete the [common prerequisites]({{< relref "common-setup.md#prerequisites" >}}), including the Alibaba Cloud CLI, and [configure its credentials](https://www.alibabacloud.com/help/en/cli/configure-credentials). To build a custom PodVM image, you also need an OSS bucket.

## ACK Preparation

### Setup cluster

1. Create ACK Managed Cluster.
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
    export SECURITY_GROUP_ID=$(aliyun cs DescribeClusterDetail --ClusterId ${CLUSTER_ID} | jq -r  ".security_group_id")
  ```
  
  Wait for the cluster to be created. Get the vSwitch id of the cluster.
  Then add one worker node to the cluster.

2. Add Internet access for the cluster VPC
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

    # The band width of the public ip (Mbps)
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
3. Grant role permissions
  Give role permission to the cluster to allow the worker to create ECS instances.

## Create the PodVM image

> **Note:**
> There is a pre-built Community Image (id:`m-j6c3kfz2vhu6ze04wacb`) for version `0.22.0` in `cn-hongkong` that you can use for testing.
> You can also use the region-specific image IDs listed under [Export PodVM image version](#export-podvm-image-version) below.

If you want to build a pod VM image yourself, follow the steps below. The current PodVM build uses [mkosi](https://github.com/systemd/mkosi) and targets **Ubuntu 26.04** (`resolute`). See the [PodVM README](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/podvm/README.md) for prerequisites and customization options.

1. Build the Ubuntu 26.04 PodVM image.
  ```bash
  cd podvm
  make
  ```

This builds the PodVM binaries and the OS image. The resulting qcow2 file is:

  ```text
  podvm/build/podvm-ubuntu-amd64.qcow2
  ```

To rebuild only the OS image after binaries are already present:

  ```bash
  cd podvm
  make image
  ```

2. Upload to OSS storage and create an ECS Image.

From `src/cloud-api-adaptor`, upload the qcow2 file to OSS (Object Storage Service):
  ```bash
  cd ..
  export REGION_ID=<region-id>
  export IMAGE_FILE=podvm/build/podvm-ubuntu-amd64.qcow2
  export BUCKET=<OSS-bucket-name>
  export OBJECT=<object-name>

  aliyun oss cp ${IMAGE_FILE} oss://${BUCKET}/${OBJECT}
  ```

Then import it as an ECS image:
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

## Deploy

### Preparation

Complete [Prepare the CAA deployment]({{< relref "common-setup.md#prepare-the-caa-deployment" >}}).

This downloads the Helm artifacts, selects the CAA image, installs cert-manager, and creates the installation namespace.

> **Note**: Keep the resulting `CAA_VERSION`, `CAA_IMAGE`, and `CAA_TAG` values in your terminal, then return here.

### Export PodVM image version

Exports the PodVM image ID used by peer pods. This variable tells the deployment tooling which PodVM image version
to use when creating peer pod virtual machines in Alibaba Cloud.

```bash
export IMAGEID="m-j6c3kfz2vhu6ze04wacb"
```

> **Note:** Alibaba Cloud builds the images ahead of time. Different regions has different image id to use.
>
> | region | IMAGEID |
> |---|---|
> | cn-beijing | m-2ze2vxvrxsbue3sf8b02 |
> | cn-hongkong | m-j6c3kfz2vhu6ze04wacb |
> | cn-hangzhou | m-bp146ws6x3iyuwjvbdw2 |
> | ap-southeast-1 | m-t4ng1w8ipua3c0o57hor |

### Select peer-pods machine type

```bash
export PODVM_INSTANCE_TYPE="ecs.g8i.xlarge"
export DISABLECVM="false"
```

> **Note:** See [the official document](https://help.aliyun.com/en/ecs/user-guide/build-a-tdx-confidential-computing-environment?spm=a2c4g.11186623.0.0.219e6187iTmYIJ#31e97c05ee64f) for more instance types that supports confidential computing.


### Populate the provider file

List of all available configuration options can be found in two places:
- [Main charts values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/values.yaml)
- [Alibaba Cloud specific values](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/alibabacloud.yaml)

Run the following command to update the [`providers/alibabacloud.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/alibabacloud.yaml) file:

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

> **Note:** If you are not using RRSA for auth, please change the `alibabacloud.rrsa.enable` to `false`
> in the yaml.

### Deploy helm chart

1. Create the secret using `kubectl`:

    See [providers/alibabacloud-secrets.yaml.template](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/charts/peerpods/providers/alibabacloud-secrets.yaml.template) for required keys.

    > **Note**: Below example assumes that you are using RRSA for auth hence
    > `ALIBABACLOUD_ACCESS_KEY_ID` and `ALIBABACLOUD_ACCESS_KEY_SECRET` are not provided,
    > while `ALIBABA_CLOUD_ROLE_ARN` and `ALIBABA_CLOUD_OIDC_PROVIDER_ARN` are provided.

    ```bash
    kubectl create secret generic my-provider-creds \
    -n confidential-containers-system \
    --from-literal=ALIBABA_CLOUD_ROLE_ARN=${ALIBABA_CLOUD_ROLE_ARN} \
    --from-literal=ALIBABA_CLOUD_OIDC_PROVIDER_ARN=${ALIBABA_CLOUD_OIDC_PROVIDER_ARN} \
    --from-literal=ALIBABA_CLOUD_OIDC_TOKEN_FILE=/var/run/secrets/ack.alibabacloud.com/rrsa-tokens/token
    ```
   
2. Install the Helm chart:

   The `-f` and `--set` customization options are described in [advanced configuration]({{< relref "../../getting-started/installation/advanced_configuration.md" >}}).

    ```bash
    helm install peerpods . \
      -f providers/alibabacloud.yaml \
      --set secrets.mode=reference \
      --set secrets.existingSecretName=my-provider-creds \
      --dependency-update \
      -n confidential-containers-system
    ```

Generic Peer pods Helm charts deployment instructions are also described 
[here](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/install/charts/peerpods/README.md).

### Verify deployment

Follow [Verify the installation and run a sample workload]({{< relref "../verify-cleanup.md#verify-the-installation" >}}), then confirm that Alibaba Cloud created the PodVM:

You can verify that the peer-pod VM was created by running the following command:

```bash
aliyun ecs DescribeInstances --RegionId ${REGION_ID} --InstanceName 'podvm-*'
```

The output should include the VM associated with the `nginx` pod.
If you run into problems, see the [troubleshooting guide]({{< relref "../../troubleshooting/_index.md" >}}).

## Uninstall

First [remove the workloads and uninstall CAA]({{< relref "../verify-cleanup.md#remove-the-sample-workload" >}}).

Verify that all peer-pod VMs are deleted:

```bash
aliyun ecs DescribeInstances --RegionId ${REGION_ID} --InstanceName 'podvm-*'
```

Delete the ACK cluster by running the following command:

```bash
aliyun cs DELETE /clusters/${CLUSTER_ID} --region ${REGION_ID} --keep_slb false --retain_all_resources false --header "Content-Type=application/json;" --body "{}"
```

