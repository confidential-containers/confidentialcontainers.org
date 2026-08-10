---
title: 工作负载加固
description: 为生产环境配置合适的 RuntimeClass 和策略
weight: 35
categories:
- getting-started
---

现在你已经部署了一个简单的 Confidential Containers 工作负载，
接下来可以进一步了解如何将其加固到适用于生产环境的水平。
本页重点介绍你需要做出的几个关键决策：

1. 为你的硬件选择合适的 RuntimeClass
2. 理解并配置用于保护工作负载的各类策略
3. 利用额外的安全特性进一步提升防护能力

## 选择合适的 RuntimeClass

在前面的示例中，我们使用的是 `kata-qemu-coco-dev`，
它主要用于测试场景，不依赖机密计算硬件支持。
如果是在生产环境中部署，则必须选择与实际 TEE 硬件相匹配的 RuntimeClass。

### RuntimeClass 选择指南

**用于开发与测试：**
- `kata-qemu-coco-dev` - 不依赖 TEE 硬件的测试运行时（⚠️ 不提供安全保证）

**用于 x86_64 裸金属生产环境：**
- `kata-qemu-tdx` - Intel TDX（Trust Domain Extensions）
- `kata-qemu-snp` - AMD SEV-SNP（Secure Encrypted Virtualization）
- `kata-qemu-sev` - AMD SEV（较早一代）
- `kata-qemu-nvidia-gpu-tdx` - NVIDIA CC GPU 加 Intel TDX 组合
- `kata-qemu-nvidia-gpu-snp` - NVIDIA CC GPU 加 AMD SNP 组合

**用于 s390x 生产环境：**
- `kata-qemu-se` - IBM Secure Execution

**用于云上部署（Peer Pods）：**
- `kata-remote` - 面向 AWS、Azure、GCP 等平台的 Cloud API Adaptor

### 示例：迁移到生产环境配置

下面展示如何将 Pod 更新为使用真实的 TEE 硬件。

**以 Intel TDX 为例：**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-production
spec:
  runtimeClassName: kata-qemu-tdx
  containers:
  - image: bitnami/nginx:1.22.0
    name: nginx
```

{{% alert title="说明" color="info" %}}
设置 `runtimeClassName` 字段通常已经足够。
有些示例为了兼容较旧配置，还会保留 `io.containerd.cri.runtime-handler` 注解，
但在使用 RuntimeClass 时，这个注解通常是冗余的。
{{% /alert %}}

**使用Intel TDX 加 NVIDIA Hopper CC GPU：**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vectoradd-kata
  namespace: default
spec:
  runtimeClassName: kata-qemu-nvidia-gpu-tdx
  restartPolicy: Never
  containers:
  - name: cuda-vectoradd
    image: "nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda12.5.0-ubuntu22.04"
    resources:
      limits:
        nvidia.com/pgpu: "1"
        memory: 16Gi
```

关于 Hopper、Blackwell 多 GPU 示例，以及 Hopper PPCIE 标签配置，请参见
[NVIDIA GPU 示例](../../examples/nvidia-gpu-examples/)。

{{% alert title="说明" color="info" %}}
所选 RuntimeClass 必须与硬件能力匹配。
如果 RuntimeClass 与实际硬件不一致（例如在 AMD 硬件上使用 `kata-qemu-tdx`），
Pod 创建将会失败。
{{% /alert %}}

## 理解 CoCo 策略体系

Confidential Containers 使用**三类策略**，分别在不同层面保护工作负载。
若要安全地部署到生产环境，理解这三类策略至关重要。

### 三类策略

| 策略类型 | 执行位置 | 控制内容 | 配置方式 |
| -------- | -------- | -------- | -------- |
| **Kata Agent 策略** | TEE 内部，由 Kata Agent 执行 | 控制 Agent 可执行的操作（如创建容器、对 Pod 执行 `exec` 等） | 通过包含 init-data 的 Pod 注解配置 |
| **KBS 资源策略** | 由 Trustee KBS 执行 | 控制哪些机密信息可以释放给哪些工作负载 | 通过 KBS Client 或 Trustee Operator 配置 |
| **证明服务策略** | 由 Trustee AS 执行 | 控制如何评估硬件证据（例如接受哪些 TCB 状态） | 通过 KBS Client 或 Trustee Operator 配置 |

{{< figure src="/img/CoCoMeasurementsAndConfig.svg" alt="展示 kata agent 策略、KBS 资源策略和证明服务策略如何在证明流程中协同工作的示意图" >}}

{{% alert title="说明" color="info" %}}
图中展示的 Rego 文件名仅为示例。
这些名称本身没有特殊含义。
实际配置时，策略始终是通过命令行参数（例如 `kbs-client ... set-resource-policy --policy-file <path>`）
或配置项按路径传入，服务实际使用的是**文件内容**，而不是文件名。
{{% /alert %}}

### 1. Kata Agent 策略（TEE 内）

Kata Agent 策略决定了 Kata Agent 在 TEE 内允许执行哪些操作。
这是防御恶意或已被入侵的 Kubernetes 控制平面的**第一道防线**。

**典型用途包括：**
- 禁止对生产 Pod 执行 `kubectl exec`
- 限制允许启动的容器镜像
- 控制允许执行的命令

下面是一个**较为严格的** Kata Agent 策略示例：

```rego
package agent_policy
import rego.v1

default CreateContainerRequest := false
default ExecProcessRequest := false

# Only allow specific image digests
CreateContainerRequest if {
	input.storages[0].source == "docker.io/library/nginx@sha256:abc123..."
}
```

Kata Agent 策略会嵌入到 Init-Data 配置文件中。
该文件还可提供其他配置，例如 Trustee 的地址信息。

**延伸阅读：**[Agent Policies and Init-Data](../../features/initdata/)

### 2. KBS 资源策略（KBS 侧）

KBS 资源策略用于控制在何种条件下释放哪些机密信息。
它会检查工作负载提交的证明令牌，并据此做出决策。

**典型用途包括：**
- 验证工作负载是否使用了特定的 Kata Agent 策略（通过 Init-Data 哈希）
- 仅向通过 TDX 远程证明的TDVM实例释放数据库凭据
- 要求特定的信任级别（例如 `affirming` 或 `contraindicated`）
- 为不同平台（TDX 或 SNP）下发不同的机密信息

**示例：校验 Init-Data 哈希**

当你在 Pod 中提供 Init-Data（其中包含 Kata Agent 策略）时，
证明服务会对其进行校验，并将相应哈希写入令牌。
你的 KBS 资源策略可以进一步验证这个特定的 Init-Data 哈希，
从而确保实际使用的是期望的 Kata Agent 策略与配置。

```rego
package policy
import rego.v1

default allow = false

# Only release secrets to workloads with the expected Init-Data hash
allow if {
	input["submods"]["cpu0"]["ear.status"] == "affirming"
	# Verify the specific Init-Data hash (includes kata agent policy + config)
	input["submods"]["cpu0"]["ear.veraison.annotated-evidence"]["init_data"] == "expected-hash-here"
}
```

请使用你在 `initdata.toml` 中指定的哈希算法计算期望值。
例如，对 TDX 场景，通常会指定 `sha384`，此时可在命令行中执行：

```bash
sha384sum initdata.toml
```

**延伸阅读：**[KBS Resource Policies](../../attestation/policies/#kbs-resource-policies)

### 3. 证明服务策略（Attestation Service 侧）

证明服务策略定义了**如何评估硬件证据**，包括：
接受哪些测量值、与哪些参考值进行比对，以及如何计算信任向量。

**典型用途包括：**
- 定义可接受的固件版本
- 为不同工作负载指定所需的安全级别
- 将硬件测量值映射为可信声明

**延伸阅读：**[Attestation Service Policies](../../attestation/policies/#attestation-service-policies)

{{% alert title="默认策略" color="success" %}}
CoCo 已为 TDX、SNP 实例以及
NVIDIA 机密计算 GPU 提供了合理的默认证明策略。
对大多数用户而言，只需补充参考值即可，策略本身通常已具备合适配置。
{{% /alert %}}

## 额外的安全特性

完成基础配置后，可以进一步了解以下功能以增强整体安全性：
[功能概览](../../features/)

## 生产环境快速检查清单

在部署到生产环境之前，请确认已完成以下事项：

- [ ] 已根据实际硬件选择正确的 RuntimeClass
- [ ] 已生成并嵌入适合当前工作负载的 Kata Agent 策略
- [ ] 已在 KBS 中配置 KBS 资源策略
- [ ] 已向证明服务预置参考值

## 后续步骤

- **部署 Trustee：** 参见[Trustee 安装](../../attestation/installation/)，以启用远程证明
- **深入策略体系：** 参见[各类策略说明](../../attestation/policies/)
- **云上部署：** 参见[AWS、Azure、GCP 云平台示例](../../examples/)