---
title: AWS
description: Cloud API Adaptor (CAA) on AWS
categories:
- examples
tags:
- caa
- aws
- eks
---

This documentation will walk you through setting up CAA (a.k.a. Peer Pods) on AWS Elastic Kubernetes Service (EKS). It explains how to deploy:

- A single worker node Kubernetes cluster using Elastic Kubernetes Service (EKS)
- CAA on that Kubernetes cluster
- An Nginx pod backed by CAA pod VM

## Pre-requisites

- Install `aws` CLI [tool](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- Install `eksctl` CLI [tool](https://eksctl.io/installation/)
- Install kubectl by following the instructions [here](https://kubernetes.io/docs/tasks/tools/#kubectl).
- Ensure that the tools `curl`, `git` and `jq` are installed.

## AWS Preparation

- Set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (or `AWS_PROFILE`) and `AWS_REGION` for AWS CLI access

- Set the region:

{{< tabpane text=true right=true persist=header >}}

{{% tab header="AMD SEV-SNP" %}}

```bash
export AWS_REGION="us-east-2"
```

> **Note:** We have chose region `us-east-2` as it has AMD SEV-SNP instances as well as prebuilt pod VM images readily available.

{{% /tab %}}

{{% tab header="Non-Confidential" %}}

```bash
export AWS_REGION="us-east-2"
```

> **Note:** We have chose region `us-east-2` because it has prebuilt pod VM images readily available.

{{% /tab %}}
{{< /tabpane >}}

### Deploy Kubernetes using EKS

Make changes to the following environment variable as you see fit:

```bash
export CLUSTER_NAME="caa-$(date '+%Y%m%b%d%H%M%S')"
export CLUSTER_NODE_TYPE="m5.xlarge"
export CLUSTER_NODE_FAMILY_TYPE="Ubuntu2204"
export SSH_KEY=~/.ssh/id_rsa.pub
```

Example EKS cluster creation using the default AWS VPC-CNI

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

Wait for the cluster to be created.

### Allow required network ports

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

# agent-protocol-forwarder port
aws ec2 authorize-security-group-ingress --group-id "$EKS_CLUSTER_SG" --protocol tcp --port 15150 --cidr "$EKS_VPC_CIDR"

# vxlan port
aws ec2 authorize-security-group-ingress --group-id "$EKS_CLUSTER_SG" --protocol tcp --port 9000 --cidr "$EKS_VPC_CIDR"
aws ec2 authorize-security-group-ingress --group-id "$EKS_CLUSTER_SG" --protocol udp --port 9000 --cidr "$EKS_VPC_CIDR"
```

> **Note:**
>
> - Port `15150` is the default port for CAA to connect to the `agent-protocol-forwarder`
> running inside the pod VM.
> - Port `9000` is the VXLAN port used by CAA. Ensure it doesn't conflict with the VXLAN port
> used by the Kubernetes CNI.

## Deploy CAA

### Download the CAA deployment artifacts

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

### CAA pod VM image

{{< tabpane text=true right=true persist=header >}}
{{% tab header="**Versions**:" disabled=true /%}}

{{% tab header="Last Release" %}}

Export this environment variable to use for the peer pod VM:

```bash
export PODVM_AMI_ID="ami-0af256cec444be636"
```

{{% /tab %}}

{{% tab header="Latest Build" %}}

There are no pre-built pod VM AMI for latest builds. You'll need to follow [these instructions](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/aws#create-pod-vm-ami) to build the pod VM AMI. Once image build is finished then export image id to the environment variable `PODVM_AMI_ID`.

{{% /tab %}}

{{% tab header="DIY" %}}

If you have made changes to the CAA code that affects the pod VM image and you want to deploy those changes then follow [these instructions](https://github.com/confidential-containers/cloud-api-adaptor/tree/main/src/cloud-api-adaptor/aws#create-pod-vm-ami) to build the pod VM AMI. Once image build is finished then export image id to the environment variable `PODVM_AMI_ID`.

{{% /tab %}}

{{< /tabpane >}}

### CAA container image

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

> **Caution**: You can also use the `latest` tag but it is **not** recommended, because of its lack of version control and potential for unpredictable updates, impacting stability and reproducibility in deployments.

{{% /tab %}}

{{% tab header="DIY" %}}

If you have made changes to the CAA code and you want to deploy those changes then follow [these instructions](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/install/README.md#building-custom-cloud-api-adaptor-image) to build the container image. Once the image is built export the environment variables `CAA_IMAGE` and `CAA_TAG`.

{{% /tab %}}

{{< /tabpane >}}

### Create the AWS credentials file

```bash
cat <<EOF > install/overlays/aws/aws-cred.env
AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
EOF
```

> **Note**: The values should be without quotes

### Select peer-pods machine type

{{< tabpane text=true right=true persist=header >}}
{{% tab header="AMD SEV-SNP" %}}

```bash
export PODVM_INSTANCE_TYPE="m6a.large"
export DISABLECVM="false"
```

Find more AMD SEV-SNP machine types on [this](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snp-work.html) AWS documentation.

{{% /tab %}}

{{% tab header="Non-Confidential" %}}

```bash
export PODVM_INSTANCE_TYPE="t3.large"
export DISABLECVM="true"
```

{{% /tab %}}
{{< /tabpane >}}

### Populate the `kustomization.yaml` file

Run the following command to update the [`kustomization.yaml`](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/install/overlays/aws/kustomization.yaml) file:

```yaml
cat <<EOF > install/overlays/azure/kustomization.yaml
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
  - CLOUD_PROVIDER="aws"  
  - DISABLECVM="${DISABLECVM}"  
  - VXLAN_PORT="${VXLAN_PORT}"  
  - PODVM_AMI_ID="${PODVM_AMI_ID}"
  - PODVM_INSTANCE_TYPE="${PODVM_INSTANCE_TYPE}"  
secretGenerator:
- name: peer-pods-secret
  namespace: confidential-containers-system  
  envs:
    - aws-cred.env
```

### Deploy CAA on the Kubernetes cluster

Label the cluster nodes with `node.kubernetes.io/worker=`

```bash
for NODE_NAME in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  kubectl label node $NODE_NAME node.kubernetes.io/worker=
done
```

Deploy the coco operator. Usually it's the same version as CAA, but it can be adjusted.

```bash
export COCO_OPERATOR_VERSION="${CAA_VERSION}"
kubectl apply -k "github.com/confidential-containers/operator/config/release?ref=v${COCO_OPERATOR_VERSION}"
kubectl apply -k "github.com/confidential-containers/operator/config/samples/ccruntime/peer-pods?ref=v${COCO_OPERATOR_VERSION}"
```

Run the following command to deploy CAA:

```bash
kubectl apply -k "install/overlays/aws"
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
aws ec2 describe-instances --filters "Name=tag:Name,Values=podvm*" \
   --query 'Reservations[*].Instances[*].[InstanceId, Tags[?Key==`Name`].Value | [0]]' --output table
```

Here you should see the VM associated with the pod `nginx`.

> **Note**: If you run into problems then check the troubleshooting guide [here](../troubleshooting/).

## Cleanup

Delete all running pods using the runtimeclass `kata-remote`. You can use the following command for the same:

```bash
kubectl get pods -A -o custom-columns='NAME:.metadata.name,NAMESPACE:.metadata.namespace,RUNTIMECLASS:.spec.runtimeClassName' | grep kata-remote | awk '{print $1, $2}'
```

Verify that all peer-pod VMs are deleted. You can use the following command to list all the peer-pod VMs
(VMs having prefix `podvm`) and status:

```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=podvm*" \
--query 'Reservations[*].Instances[*].[InstanceId, Tags[?Key==`Name`].Value | [0], State.Name]' --output table
```

Delete the EKS cluster by running the following command:

```bash
eksctl delete cluster --name=$EKS_CLUSTER_NAME 
```
