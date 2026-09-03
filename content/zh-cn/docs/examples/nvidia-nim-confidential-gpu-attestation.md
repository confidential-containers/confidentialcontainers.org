---
title: NVIDIA 机密 NIM 部署
description: >-
  使用 Kata、Trustee Key Broker Service、sealed secret 和 GPU 远程证明
  完成端到端 NVIDIA NIM 部署
categories:
- examples
tags:
- gpu
- nvidia
- nim
- kbs
- attestation
---

> **说明：** 本文为英文文档的中文译版，英文原版请参见 [NVIDIA confidential NIM deployment](https://confidentialcontainers.org/docs/examples/nvidia-nim-confidential-gpu-attestation/)。

本文演示如何把 [NVIDIA NIM](https://docs.nvidia.com/nim/index.html) 推理工作负载部署到 Kubernetes，并运行在 Confidential Containers 上。这里以一台启用了 NVIDIA GPU 机密计算能力的 AMD SEV-SNP Kubernetes 工作节点为例。相同的方法也可以迁移到 Intel TDX 节点，但参考值与证明策略需要按 TDX 重新生成，本文不展开。

NVIDIA NIM 将基础模型封装成容器，并提供针对 GPU 优化的运行时与 HTTP API。本文从一个普通的 NIM Pod 清单开始，镜像为 `nvcr.io/nim/meta/llama-3.1-8b-instruct:1.13.1`，通过 chat completions API 提供 Meta Llama 3.1 8B Instruct 模型服务。可选的基线步骤会先用非机密的 `kata-qemu-nvidia-gpu` runtime class 启动它，并检查 8000 端口上的健康状态、模型列表与推理接口。机密场景则切换为 `kata-qemu-nvidia-gpu-snp` runtime class，把 Pod 放进机密虚拟机中；但仅改 runtime class 还不够，还需要补齐 Trustee 的 KBS、guest pull、AA 与 CDH 配置、sealed secret、镜像签名策略、生成的 Kata agent policy、可信存储，以及批准 CPU、GPU 与 initdata 证据的 KBS 策略。下面按 checkpoint 逐步完成这些配置。

{{% alert title="目标读者与适用范围" color="primary" %}}

这是一份面向平台与基础设施工程师的单节点动手教程，用于通过远程证明端到端验证机密 NIM 工作负载。它不是面向全集群部署的生产运维指南。文中多个步骤刻意保留为手工操作，以便逐一检查各信任边界；教程还会在 worker 节点上执行本地操作，例如创建本地 backing storage、在与工作负载同一节点上运行 Trustee，或从在线 QEMU 进程收集 SNP 启动输入。

此外，本教程在整个流程中刻意采用单一操作者角色：同一人完成平台搭建、KBS/资源配置、策略生成、观测启动数据收集以及工作负载验证。这样在单节点教程里能完整看到端到端流程，但并不反映生产环境中的职责划分。相关步骤中会标注生产注意事项，文末 [展望：生产自动化](#展望生产自动化) 总结了如何将同一流程结构化用于生产环境。

{{% /alert %}}

## 前提条件

开始前请准备：

- 一个已经部署好 Confidential Containers、Kata、NVIDIA GPU Operator，并提供机密 GPU runtime class（例如 `kata-qemu-nvidia-gpu-snp`）的 Kubernetes 集群。可参考 [NVIDIA Confidential Containers deployment guide](https://docs.nvidia.com/datacenter/cloud-native/confidential-containers/latest/confidential-containers-deploy.html)
- 可访问 Kubernetes 工作节点，因为本文需要从在线 QEMU 进程中收集 SNP 启动输入
- 一个可拉取 NIM 镜像的 [NGC API key](https://docs.nvidia.com/ngc/latest/ngc-catalog-user-guide.html#generating-a-personal-api-key)
- 工作节点上安装好 `kubectl`、`curl`、`jq`、`openssl`、`oras`、`skopeo`、`envsubst`、`base64`、`tar`、`zstd` 和 `cargo`
- 工作节点上安装 Docker 及 Docker Compose 插件
- 工作节点上安装 Python 包 `cryptography` 和 `jwcrypto`，后面生成 sealed secret 辅助脚本会用到

{{% alert title="关于制品来源" color="info" %}}
本文尽量使用已经发布或预构建的软件制品，避免读者为了验证流程再去克隆仓库或本地构建组件。只有在确实没有可直接使用的软件制品时，才会局部引入额外步骤。
{{% /alert %}}

## 流程概览

本文把机密部署拆成一组可逐步验证的 checkpoints。每完成一步，都应能看到当前状态并在继续前修正问题。

1. **可选：非机密基线验证**：把 GPU 切到非机密模式，部署基线 Pod，确认 NIM API 可用，删除 Pod，再把 GPU 切回机密模式。
2. **部署 Trustee 并安装 kbs-client**：在节点上直接部署 Trustee，选择匹配的 Trustee 版本，配置 KBS HTTPS 端点，并准备 KBS。
3. **准备 TEE NIM 清单与 sealed secret**：生成运行时密钥对应的 sealed secret，定义引用该密钥、镜像拉取 Secret 与可信存储的 SNP GPU Pod 清单。
4. **创建 Pod initdata 注解**：为 AA 和 CDH 配置 KBS 地址、KBS 证书、镜像仓库凭据 URI 以及镜像签名策略 URI。
5. **生成 Kata agent policy**：对 TEE Pod 清单运行 `genpolicy`，把策略与 initdata 合并，写回生成的 `cc_init_data` 注解，并保留生成后的清单但暂不部署。
6. **收集 SNP 启动参考值**：如果还没有现成的参考值，就用生成好的 `cc_init_data` 启动一个轻量级度量 Pod，记录 SNP launch measurement、SNP TCB 值以及已证明的 initdata 摘要，然后删除测量 Pod。
7. **写入 KBS 资源、参考值与策略**：配置 NVIDIA 镜像签名校验、公有镜像仓库凭据、NIM 运行时 API key、SNP 参考值、sealed secret 验证公钥，以及要求 CPU、GPU 证据和 initdata 摘要都匹配的 KBS 策略。
8. **准备可信存储资源**：创建 TEE NIM Pod 所依赖的可信存储。
9. **运行端到端场景**：创建 Kubernetes 侧的 Secret，部署带策略的 TEE NIM 清单，并验证在 KBS 策略约束下 NIM API 仍可正常访问。

## 初始 NIM Manifest

先导出一个可拉取 NIM 镜像的 NGC API key，然后生成初始清单模板。如果你不做 checkpoint 1 的非机密基线验证，那么这个步骤不需要实际部署。

这个 Manifest 会在两个地方使用 NGC API key：`ngc-secret-instruct` 供 Kubernetes 从 `nvcr.io` 拉取镜像，`ngc-api-key-instruct` 则把 key 传给 NIM 容器在运行时使用。为了简化示例，本文复用了同一个 key。实际环境中，建议把镜像拉取与容器运行时使用的 key 分开。

该文件是可选基线部署的基础 Manifest。它保留 `${NGC_API_KEY}` 作为占位符，后面的校验和部署命令会通过 `envsubst` 把它替换后再交给 `kubectl`。

```bash
export NGC_API_KEY="<NGC_API_KEY>"

cat <<'EOF' | tee nvidia-nim-llama-3-1-8b-instruct.yaml >/dev/null
apiVersion: v1
kind: Secret
metadata:
  name: ngc-secret-instruct
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "nvcr.io": {
          "username": "$oauthtoken",
          "password": "${NGC_API_KEY}"
        }
      }
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: ngc-api-key-instruct
type: Opaque
stringData:
  api-key: "${NGC_API_KEY}"
---
apiVersion: v1
kind: Pod
metadata:
  name: nvidia-nim-llama-3-1-8b-instruct
  labels:
    app: nvidia-nim-llama-3-1-8b-instruct
spec:
  restartPolicy: Never
  runtimeClassName: kata-qemu-nvidia-gpu
  imagePullSecrets:
    - name: ngc-secret-instruct
  containers:
    - name: nvidia-nim-llama-3-1-8b-instruct
      image: nvcr.io/nim/meta/llama-3.1-8b-instruct:1.13.1
      ports:
        - containerPort: 8000
          name: http-openai
      livenessProbe:
        httpGet:
          path: /v1/health/live
          port: http-openai
        initialDelaySeconds: 15
        periodSeconds: 10
        timeoutSeconds: 1
        successThreshold: 1
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /v1/health/ready
          port: http-openai
        initialDelaySeconds: 15
        periodSeconds: 10
        timeoutSeconds: 1
        successThreshold: 1
        failureThreshold: 3
      startupProbe:
        httpGet:
          path: /v1/health/ready
          port: http-openai
        initialDelaySeconds: 360
        periodSeconds: 10
        timeoutSeconds: 1
        successThreshold: 1
        failureThreshold: 120
      env:
        - name: NGC_API_KEY
          valueFrom:
            secretKeyRef:
              name: ngc-api-key-instruct
              key: api-key
      resources:
        limits:
          nvidia.com/pgpu: "1"
          cpu: "16"
          memory: "48Gi"
EOF
```

## Checkpoint 1：可选的非机密基线验证

如果你只关心机密部署，可以跳过这一节。这里的作用只是先验证普通 NIM Pod 在 Kata Containers 上能否正常运行。流程是：临时把 GPU 切到非机密模式，运行非 TEE Pod，验证后删除，再切回机密模式。

这里通过节点标签触发模式切换，NVIDIA GPU Operator 会根据标签完成相应编排。最后一定要把节点切回机密 GPU 模式，后续 checkpoint 才能继续。关于 GPU runtime class 和机密 GPU 模式标签，可参考 [NVIDIA GPU 示例](https://confidentialcontainers.org/zh-cn/docs/examples/nvidia-gpu-examples/)。

先确保节点处于 VM passthrough 和非机密 GPU 模式：

```bash
kubectl label nodes --all \
  nvidia.com/gpu.workload.config=vm-passthrough \
  --overwrite

kubectl label nodes --all nvidia.com/cc.mode=off --overwrite
```

切换 CC 模式时，GPU Operator 的一些组件可能会重启。先等待节点标签确认模式切换完成，再等待 GPU Operator 相关控制器稳定。这里不要直接等所有 Pod 全部就绪，因为模式切换期间 operator 可能会删除并重建一部分 Pod。

```bash
kubectl wait \
  --for=jsonpath='{.metadata.labels.nvidia\.com/cc\.mode\.state}'=off \
  node --all \
  --timeout=15m

kubectl -n gpu-operator wait \
  --for=condition=Available \
  deployment --all \
  --timeout=10m

kubectl -n gpu-operator rollout status \
  daemonset/nvidia-vfio-manager \
  --timeout=10m

kubectl -n gpu-operator rollout status \
  daemonset/nvidia-sandbox-validator \
  --timeout=10m

kubectl -n gpu-operator rollout status \
  daemonset/nvidia-kata-sandbox-device-plugin-daemonset \
  --timeout=10m

kubectl -n gpu-operator rollout status \
  daemonset/nvidia-cc-manager \
  --timeout=10m

kubectl get nodes \
  -L nvidia.com/gpu.workload.config,nvidia.com/cc.mode,nvidia.com/cc.mode.state
```

部署非机密模式：

```bash
: "${NGC_API_KEY:?set NGC_API_KEY to your NGC API key}"

envsubst '${NGC_API_KEY}' \
  < nvidia-nim-llama-3-1-8b-instruct.yaml \
  | kubectl apply -f -

kubectl wait --for=condition=Ready \
  --timeout=600s \
  pod/nvidia-nim-llama-3-1-8b-instruct
```

演示环境里，直接从能访问集群 Pod 网络的机器上查询 Pod IP，例如单节点集群宿主机。这一步不会创建 Kubernetes Service。生产环境通常会通过 Service、Ingress、Gateway 或其他受控入口暴露 NIM。

检查 NIM 是否就绪并列出模型名：

```bash
POD_IP="$(kubectl get pod nvidia-nim-llama-3-1-8b-instruct -o jsonpath='{.status.podIP}')"

curl -fsS "http://${POD_IP}:8000/v1/health/ready" | jq .
curl -fsS "http://${POD_IP}:8000/v1/models" | jq -r '.data[].id'
```

发送一个最简单的 chat completion 请求：

```bash
MODEL_NAME="$(curl -fsS "http://${POD_IP}:8000/v1/models" | jq -r '.data[0].id')"

curl -fsS "http://${POD_IP}:8000/v1/chat/completions" \
  -H 'Content-Type: application/json' \
  -d "$(jq -n --arg model "${MODEL_NAME}" '{
    model: $model,
    messages: [{role: "user", content: "Reply with exactly: hello from nim"}],
    max_tokens: 16,
    temperature: 0
  }')" | jq -r '.choices[0].message.content'
```

验证通过后，删除这个非 TEE 示例 Pod 及其 Kubernetes Secret：

```bash
kubectl delete pod nvidia-nim-llama-3-1-8b-instruct --ignore-not-found
kubectl delete secret ngc-secret-instruct ngc-api-key-instruct \
  --ignore-not-found
```

把节点切回机密 GPU 模式：

```bash
kubectl label nodes --all nvidia.com/cc.mode=on --overwrite
```

然后重复上面的 CC 模式稳定性检查，只是这次等待 `nvidia.com/cc.mode.state` 变成 `on`。确认节点报告 `cc.mode=on` 和 `cc.mode.state=on` 后再继续。

## Checkpoint 2：部署 Trustee 并安装 kbs-client

这一节会把 Trustee 直接部署在节点上，后续用于写入 KBS 资源、参考值和资源策略的管理工具 [`kbs-client`](https://confidentialcontainers.org/docs/attestation/client-tool/)。KBS 是 Trustee 暴露给 Guest 侧的证明与资源服务，同时也提供 `kbs-client` 使用的管理接口。

Trustee 可以通过 Docker Compose、Helm Chart、Kubernetes 清单或其他方式部署。本文选择 Docker Compose，是因为它能直接从 Trustee 源码树同时启动 KBS、Attestation Service（AS）和 Reference Value Provider Service（RVPS）。在这个 Compose 部署中，KBS、AS 和 RVPS 是三个独立容器：KBS 调用 AS 验证证据，AS 再调用 RVPS 获取参考值。Compose 文件里还会有一个一次性运行的 `setup` 服务，用来生成演示所需的部分密钥材料。

把 Trustee 跑在与工作负载同一台单节点机器上只是为了演示方便，并不代表推荐的生产部署方式。生产环境应把 Trustee 部署在独立的可信环境中，给 KBS 使用持久化存储，使用受信任 CA 签发的 HTTPS 证书，并严格限制管理接口访问。其他部署方式可参考 [Trustee installation](https://confidentialcontainers.org/zh-cn/docs/attestation/installation/) 文档。

### 设置 KBS 部署参数

先定义本地工作目录、Guest 组件访问 KBS 时使用的 URL，以及 Compose 部署派生的各个路径。Docker Compose 会把 KBS 端口发布到工作节点主机上。由于这里是单节点集群，Guest 可以直接通过节点 IP 和宿主机端口访问 KBS：

```bash
# Local paths.
KBS_WORKDIR="${HOME}/nim-kbs"
KBS_TRUSTEE_DIR="${KBS_WORKDIR}/trustee"
KBS_CONFIG_DIR="${KBS_TRUSTEE_DIR}/kbs/config"
KBS_COMPOSE_CONFIG_DIR="${KBS_CONFIG_DIR}/docker-compose"
KBS_DATA_DIR="${KBS_TRUSTEE_DIR}/kbs/data"
KBS_STORAGE_DIR="${KBS_DATA_DIR}/kbs-storage"

# Artifact selection.
KBS_CLIENT_ARCH="${KBS_CLIENT_ARCH:-x86_64}"

# Network endpoint.
KBS_PORT="${KBS_PORT:-8080}"
KBS_NODE_IP="$(kubectl get nodes \
  -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')"
KBS_URL="https://${KBS_NODE_IP}:${KBS_PORT}"

mkdir -p "${KBS_WORKDIR}"
```

这些路径基本都由 `KBS_WORKDIR` 派生。通常只需要决定 `KBS_WORKDIR`、`KBS_PORT`，以及可选的 `KBS_CLIENT_ARCH`：

- `KBS_TRUSTEE_DIR`：下载后的 Trustee 源码
- `KBS_CONFIG_DIR`：Compose 服务挂载使用的演示配置和身份信息目录
- `KBS_COMPOSE_CONFIG_DIR`：Compose 部署使用的 KBS TOML 配置文件目录
- `KBS_DATA_DIR`：Compose 管理的演示状态目录，RVPS 参考值和 AS 状态都保存在它的子目录中
- `KBS_STORAGE_DIR`：KBS 的资源与资源策略仓库存储位置

后面会把 `KBS_URL` 写进 Pod 的 initdata，供来 Guest 内的 AA 与 CDH 访问 KBS。当前 Compose 方案中，KBS 监听 `KBS_PORT` 暴露出的宿主机端口，Kata Guest 通过节点 IP 到达该端点。生产环境里，KBS 一般位于独立可信环境，并通过稳定的服务地址暴露；Guest 网络也需要确保能够访问该地址。

生产环境中，建议根据 Trustee 所在信任边界使用合适的持久化存储，例如专用持久卷或可信外部后端。

### 选择 Trustee 部署制品

这一步要选定 Trustee 源码、Compose 服务镜像，以及与之匹配的 `kbs-client` 软件制品。原则上，应使用与你当前 CoCo 版本和 Guest 组件匹配、已经验证过的 Trustee 版本。NVIDIA 的 supported-platforms 文档通常是查这个映射关系的权威来源，可参考 [NVIDIA supported platforms](https://docs.nvidia.com/datacenter/cloud-native/confidential-containers/latest/supported-platforms.html)。

如果你的 Kata 是通过 `kata-deploy` Helm chart 安装的，通常会在 `/opt/kata/versions.yaml` 中携带发布元数据。下面的命令会自动读取其中的 `coco-trustee` 项并解析出匹配的 Trustee 版本。如果这个文件不可用，则需要到对应 Kata 版本的上游 [`versions.yaml`](https://github.com/kata-containers/kata-containers/blob/main/versions.yaml) 查找 `externals.coco-trustee.version`。按当前版本约定，Trustee 的版本号通常比 CoCo 小一个 minor，例如 CoCo `v0.18.0` 对应 Trustee `v0.17.0`。如果仍然无法解析，请手工设置 `KBS_TRUSTEE_REF`。

这种方式很适合本教程，因为相关命令会直接在安装 Kata 的节点上运行。但在生产环境中，应将版本选择纳入部署流程。如果你已经确定要使用的 Trustee 引用，请在运行以下代码片段前设置  `KBS_TRUSTEE_REF`，这样即可跳过自动查找过程：

```bash
KATA_VERSIONS_FILE="${KATA_VERSIONS_FILE:-/opt/kata/versions.yaml}"
KBS_TRUSTEE_REF="${KBS_TRUSTEE_REF:-}"
KBS_IMAGE_REPO="ghcr.io/confidential-containers/staged-images/kbs-grpc-as"
KBS_AS_IMAGE_REPO="ghcr.io/confidential-containers/staged-images/coco-as-grpc"
KBS_RVPS_IMAGE_REPO="ghcr.io/confidential-containers/staged-images/rvps"
KBS_CLIENT_IMAGE_REPO="ghcr.io/confidential-containers/staged-images/kbs-client"

if test -z "${KBS_TRUSTEE_REF}"; then
  if ! test -r "${KATA_VERSIONS_FILE}"; then
    echo "ERROR: cannot resolve KBS_TRUSTEE_REF because ${KATA_VERSIONS_FILE} is not readable." >&2
    echo "Look up the validated Trustee reference for your platform, set KBS_TRUSTEE_REF, and rerun this snippet." >&2
    exit 1
  fi

  KBS_TRUSTEE_REF="$(
    awk '
      $1 == "coco-trustee:" { in_trustee = 1; next }
      in_trustee && $1 == "version:" {
        gsub(/"/, "", $2)
        print $2
        exit
      }
    ' "${KATA_VERSIONS_FILE}"
  )"
fi

if test -z "${KBS_TRUSTEE_REF}"; then
  echo "ERROR: could not find the coco-trustee version in ${KATA_VERSIONS_FILE}." >&2
  echo "Look up the validated Trustee reference for your platform, set KBS_TRUSTEE_REF, and rerun this snippet." >&2
  exit 1
fi

KBS_COMPOSE_IMAGE_TAG="${KBS_COMPOSE_IMAGE_TAG:-${KBS_TRUSTEE_REF}-${KBS_CLIENT_ARCH}}"
KBS_CLIENT_ARTIFACT="${KBS_CLIENT_IMAGE_REPO}:sample_only-${KBS_COMPOSE_IMAGE_TAG}"
KBS_TRUSTEE_TARBALL_URL="${KBS_TRUSTEE_TARBALL_URL:-https://github.com/confidential-containers/trustee/archive/${KBS_TRUSTEE_REF}.tar.gz}"

skopeo inspect "docker://${KBS_IMAGE_REPO}:${KBS_COMPOSE_IMAGE_TAG}" >/dev/null
skopeo inspect "docker://${KBS_AS_IMAGE_REPO}:${KBS_COMPOSE_IMAGE_TAG}" >/dev/null
skopeo inspect "docker://${KBS_RVPS_IMAGE_REPO}:${KBS_COMPOSE_IMAGE_TAG}" >/dev/null
oras manifest fetch "${KBS_CLIENT_ARTIFACT}" >/dev/null

printf 'KBS_TRUSTEE_REF=%s\nKBS_IMAGE=%s:%s\nAS_IMAGE=%s:%s\nRVPS_IMAGE=%s:%s\nKBS_CLIENT_ARTIFACT=%s\nKBS_URL=%s\n' \
  "${KBS_TRUSTEE_REF}" \
  "${KBS_IMAGE_REPO}" \
  "${KBS_COMPOSE_IMAGE_TAG}" \
  "${KBS_AS_IMAGE_REPO}" \
  "${KBS_COMPOSE_IMAGE_TAG}" \
  "${KBS_RVPS_IMAGE_REPO}" \
  "${KBS_COMPOSE_IMAGE_TAG}" \
  "${KBS_CLIENT_ARTIFACT}" \
  "${KBS_URL}"
```

如果你手动将 `KBS_TRUSTEE_REF` 指定为某个发布版本标签或其他源码引用，并且 Compose 镜像使用了不同的标签命名方式，那么还应同时设置 `KBS_COMPOSE_IMAGE_TAG`。默认情况下，上述配置假定 Compose 镜像标签与 Kata 发布元数据所引用的预构建镜像保持一致。

此外，如果对应的源码树无法通过默认的 GitHub Archive URL 获取，还需要额外设置 `KBS_TRUSTEE_TARBALL_URL`，以指定正确的源码压缩包地址。

### 准备 Trustee Compose 部署

下载与所选版本匹配的 Trustee 源码 tarball，后面会直接复用其中的 Docker Compose 逻辑和 `kbs/config` 下的配置文件：

```bash
rm -rf "${KBS_TRUSTEE_DIR}"
mkdir -p "${KBS_TRUSTEE_DIR}"

curl -fsSL "${KBS_TRUSTEE_TARBALL_URL}" |
  tar -xz --strip-components=1 -C "${KBS_TRUSTEE_DIR}"
```

这一节会用到以下输入文件：

- `docker-compose.yml`：启动 KBS、AS、RVPS，以及一次性的 `setup` 服务，用于生成演示环境需要的管理密钥和 AS token 签名密钥
- `kbs/config/docker-compose/kbs-config.toml`：配置 KBS 端点、管理认证和资源存储
- `kbs/config/as-config.json`：配置 AS，包括它如何访问 RVPS 以及 NVIDIA verifier
- `kbs/config/rvps.json`：配置 RVPS

本文不逐行展开这些配置；如果你想看默认值，可直接检查下载的源码或 Trustee 仓库中的相同路径。

上游 Compose 文件默认使用 `latest` 镜像标签。下面把 KBS、AS、RVPS 镜像固定到上面解析出的 Trustee 版本，并把 KBS 暴露到 `KBS_PORT`：

```bash
sed -i \
  -e "s#image: ghcr.io/confidential-containers/staged-images/kbs-grpc-as:latest#image: ${KBS_IMAGE_REPO}:${KBS_COMPOSE_IMAGE_TAG}#" \
  -e "s#image: ghcr.io/confidential-containers/staged-images/coco-as-grpc:latest#image: ${KBS_AS_IMAGE_REPO}:${KBS_COMPOSE_IMAGE_TAG}#" \
  -e "s#image: ghcr.io/confidential-containers/staged-images/rvps:latest#image: ${KBS_RVPS_IMAGE_REPO}:${KBS_COMPOSE_IMAGE_TAG}#" \
  -e "s#\"8080:8080\"#\"${KBS_PORT}:8080\"#" \
  "${KBS_TRUSTEE_DIR}/docker-compose.yml"
```

Compose 中的 `setup` 服务会在启动时生成演示用的管理员密钥对和 AS token 签名材料，但它不会生成 KBS HTTPS 证书。因此这里还要补上 `kbs-https.*` 证书，并把它接到 KBS 配置里。证书的 `subject alternative name` 必须覆盖 `KBS_URL` 中使用的节点 IP，这样Guest组件和 `kbs-client` 才能正确校验证书：

```bash
openssl req -x509 \
  -out "${KBS_CONFIG_DIR}/kbs-https.crt" \
  -keyout "${KBS_CONFIG_DIR}/kbs-https.key" \
  -newkey rsa:2048 \
  -nodes \
  -sha256 \
  -days 365 \
  -subj "/CN=${KBS_NODE_IP}" \
  -addext "subjectAltName=IP:${KBS_NODE_IP},IP:127.0.0.1,DNS:localhost" \
  -addext "basicConstraints=CA:FALSE"

sed -i \
  -e 's#insecure_http = true#insecure_http = false\
private_key = "/opt/confidential-containers/kbs/user-keys/kbs-https.key"\
certificate = "/opt/confidential-containers/kbs/user-keys/kbs-https.crt"#' \
  "${KBS_COMPOSE_CONFIG_DIR}/kbs-config.toml"

grep -q 'insecure_http = false' "${KBS_COMPOSE_CONFIG_DIR}/kbs-config.toml"
grep -q 'type = "Simple"' "${KBS_COMPOSE_CONFIG_DIR}/kbs-config.toml"
jq -e '.verifier_config.nvidia_verifier.type == "Remote"' \
  "${KBS_CONFIG_DIR}/as-config.json" >/dev/null
```

这个演示环境里会涉及三组身份材料：

- `kbs-https.key` 与 `kbs-https.crt`：标识 KBS HTTPS 端点
- `private.key` 与 `public.pub`：用于 `kbs-client` 调用管理 API 的认证。KBS 配置中使用公钥，`kbs-client` 用私钥签名请求。这对密钥由 Compose 的 `setup` 服务生成
- `ca.key`、`ca-cert.pem`、`token.key`、`token-cert.pem`、`token-cert-chain.pem`：AS 用它们签发 KBS 可验证的 attestation token。这部分同样由 `setup` 服务生成

在当前单节点演示里，KBS 和 AS 一起启动，所以最后一组材料并不体现独立运营边界；但在 KBS 与 AS 分开部署时，它们就非常重要了。

生产环境里要明确这些身份信息的归属和保护方式。例如：KBS 证书通常应由组织 CA 签发；管理签名密钥应由正式身份流程发放和保护；AS token 签名私钥可能需要 HSM 或托管密钥服务来托管。

### 部署 Trustee

启动 Compose 部署。`--project-directory` 用于确保 `docker-compose.yml` 里的相对路径都在下载后的 Trustee 源码内解析。由于 Compose 文件本身也包含本地开发用的 `build:` 段，下面会先拉取固定版本镜像，再用 `--no-build` 避免在教程节点上本地构建：

```bash
docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  pull kbs as rvps setup

docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  up -d --no-build
```

检查 Compose 服务状态。这个版本的 Trustee 没有独立 health endpoint，因此功能性验证会放到安装完 `kbs-client` 之后再做：

```bash
docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  ps
```

保留好 `KBS_URL`、管理员 token 和 KBS HTTPS 证书，后续 checkpoint 还会继续用到：

```bash
KBS_ADMIN_TOKEN_FILE="${KBS_CONFIG_DIR}/admin-token"
KBS_CERT_FILE="${KBS_CONFIG_DIR}/kbs-https.crt"
test -s "${KBS_ADMIN_TOKEN_FILE}"
test -s "${KBS_CERT_FILE}"
```

### 安装 kbs-client

接下来安装用于 KBS 管理操作的 `kbs-client`。上面的变量 `KBS_CLIENT_ARTIFACT` 指向与选定 Trustee 版本匹配的预构建客户端制品。下面的 `oras pull` 会把它解包到 `${KBS_WORKDIR}/bin` 中。

这样可以确保客户端与正在运行的 KBS 服务版本以及管理认证模式匹配。这里还不会修改 KBS 状态；真正的资源写入在后面的 checkpoint 才发生。

```bash
command -v oras >/dev/null

mkdir -p "${KBS_WORKDIR}/bin"

oras pull \
  --output "${KBS_WORKDIR}/bin" \
  "${KBS_CLIENT_ARTIFACT}"

chmod +x "${KBS_WORKDIR}/bin/kbs-client"
KBS_CLIENT="${KBS_WORKDIR}/bin/kbs-client"

"${KBS_CLIENT}" --version
```

用一个不含敏感信息的探测资源写入再删除，验证 HTTPS 端点和管理员认证是否正常：

```bash
printf 'checkpoint2-probe\n' > "${KBS_WORKDIR}/checkpoint2-probe.txt"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-resource \
  --path default/checkpoint2/probe \
  --resource-file "${KBS_WORKDIR}/checkpoint2-probe.txt"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  delete-resource \
  --path default/checkpoint2/probe
```

后面继续操作时，保留当前 shell，或者把 `KBS_CLIENT`、`KBS_WORKDIR`、`KBS_ADMIN_TOKEN_FILE`、`KBS_CERT_FILE` 和 `KBS_URL` 私下保存好。不要删除 `KBS_WORKDIR`，因为里面包含 Trustee 源码、Compose 配置、演示状态以及后续生成的所有教程文件。由于后续 `kbs-client` 会使用 Compose 生成的管理员 token 调用 KBS，请把它当成管理员凭据妥善保护。

## Checkpoint 3：准备 TEE NIM 清单与 sealed secret

这一节会先为运行时 NGC API key 生成一个签名过的 [sealed secret](https://confidentialcontainers.org/zh-cn/docs/features/sealed-secrets/)，再生成 TEE NIM Pod 清单，但此时还不部署 Pod。

这个清单会使用 SNP GPU runtime class，引用一个 Kubernetes 镜像拉取 Secret、一个承载 sealed runtime secret 的 Kubernetes Secret、一个 [可信存储](https://confidentialcontainers.org/zh-cn/docs/features/protected-storage/) PVC，以及沿用基准清单中定义的健康检查配置。镜像拉取 Secret 与 PVC 会在后面真正部署前再创建。

之所以在部署前先生成清单，是因为 `genpolicy` 需要读取 Pod 规格作为输入。后面生成的 Kata agent policy 和 `cc_init_data` 都依赖这一版清单，测量 checkpoint 也会复用同一份 `cc_init_data`。

这里还启用了可信镜像存储。由于 guest pull 会在Guest内下载较大的 NIM 模型数据和镜像层，如果全部放在Guest的内存型文件系统中，会迫使机密虚拟机配置更大的内存。把这些数据放到挂载进机密虚拟机的块设备上，会更符合这个场景。

这里有两个和 NGC 相关的 Secret：

- `ngc-secret-instruct`：镜像拉取 Secret，供 Kubernetes 和 CRI 访问认证过的仓库元数据。生产环境中应尽量缩小 token 权限范围，并评估是否能避免把高权限凭据暴露给工作负载集群控制面。这个 Secret 稍后创建。
- `ngc-api-key-sealed-instruct`：只保存签名后的 sealed 值。Kubernetes 看到的是 sealed 值；CDH 会在 Guest 内通过从 KBS 读取 `kbs:///default/ngc-api-key/instruct` 来解密。该 Secret 之所以已经出现在这里，是因为 `genpolicy` 会在 YAML 输入中解析 `secretKeyRef`。

先使用 P-256 生成 sealed secret 签名密钥。私钥 JWK 保留在 KBS 之外，后面只上传公钥 JWK。CDH 会用这个公钥来验证 sealed NIM 运行时 secret 的签名，确认其确实来自期望的密钥，然后再去 KBS 取出对应的明文 secret。生产环境中，这类密钥应在可信发布流水线中生成和保护，只公开经过审核的 sealed 值。

```bash
KBS_WORKDIR="${KBS_WORKDIR}" python3 - <<'EOF'
import os
from jwcrypto import jwk

workdir = os.environ["KBS_WORKDIR"]
k = jwk.JWK.generate(
    kty='EC', crv='P-256', alg='ES256',
    use='sig', kid='sealed-secret-nim-key')
open(f'{workdir}/signing-key-private.jwk', 'w').write(k.export_private())
open(f'{workdir}/signing-key-public.jwk', 'w').write(k.export_public())
EOF
```

接着生成一个指向 KBS 中 NIM 运行时 API key 的签名 vault sealed secret。正常情况下，应使用 Confidential Containers 项目的 `secret` CLI 来完成。但由于本文尽量避免要求读者从源码构建 CoCo 组件，而 `secret` CLI 在写作时还没有单独发布制品，所以这里临时用一个兼容脚本生成等价的 vault sealed-secret 格式。生产环境不要直接复用这段脚本，应改用正式发布并受支持的 CoCo 工具链。

vault sealed secret 本质上是一个 JWS 签名的 JSON 文档，其中包含 KBS 资源 URI 和 provider 元数据；受保护 header 里的 `b64` 字段是为了与 `secret` CLI 产出的格式保持一致。

```bash
KBS_WORKDIR="${KBS_WORKDIR}" python3 - <<'EOF'
import base64
import json
import os
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec, utils

workdir = os.environ["KBS_WORKDIR"]
signing_jwk = json.loads(open(f"{workdir}/signing-key-private.jwk").read())

def b64url_decode(value):
    return base64.urlsafe_b64decode(value + "=" * (-len(value) % 4))

def b64url_encode(value):
    return base64.urlsafe_b64encode(value).rstrip(b"=").decode()

payload = {
    "version": "0.1.0",
    "type": "vault",
    "name": "kbs:///default/ngc-api-key/instruct",
    "provider": "kbs",
    "provider_settings": {},
    "annotations": {},
}

protected = {
    "b64": True,
    "alg": "ES256",
    "kid": "kbs:///default/signing-key/sealed-secret",
}

private_value = int.from_bytes(b64url_decode(signing_jwk["d"]), "big")
private_key = ec.derive_private_key(private_value, ec.SECP256R1())

protected_b64 = b64url_encode(json.dumps(
    protected, separators=(",", ":")).encode())
payload_b64 = b64url_encode(json.dumps(payload, separators=(",", ":")).encode())
signing_input = f"{protected_b64}.{payload_b64}".encode()
signature_der = private_key.sign(signing_input, ec.ECDSA(hashes.SHA256()))
r, s = utils.decode_dss_signature(signature_der)
signature = r.to_bytes(32, "big") + s.to_bytes(32, "big")

with open(f"{workdir}/ngc-api-key-instruct.sealed", "w") as f:
    f.write(f"sealed.{protected_b64}.{payload_b64}.{b64url_encode(signature)}")
EOF
```

生成 Pod Manifest：

```bash
NIM_TEE_MANIFEST="${KBS_WORKDIR}/nvidia-nim-llama-3-1-8b-instruct-tee.yaml"

SEALED_NGC_API_KEY_BASE64="$(
  base64 -w0 "${KBS_WORKDIR}/ngc-api-key-instruct.sealed"
)"

cat > "${NIM_TEE_MANIFEST}" <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: ngc-api-key-sealed-instruct
type: Opaque
data:
  api-key: "${SEALED_NGC_API_KEY_BASE64}"
---
apiVersion: v1
kind: Pod
metadata:
  name: nvidia-nim-llama-3-1-8b-instruct
  labels:
    app: nvidia-nim-llama-3-1-8b-instruct
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    supplementalGroups: [4, 20, 24, 25, 27, 29, 30, 44, 46]
  restartPolicy: Never
  runtimeClassName: kata-qemu-nvidia-gpu-snp
  imagePullSecrets:
    - name: ngc-secret-instruct
  containers:
    - name: nvidia-nim-llama-3-1-8b-instruct
      image: nvcr.io/nim/meta/llama-3.1-8b-instruct:1.13.1
      ports:
        - containerPort: 8000
          name: http-openai
      livenessProbe:
        httpGet:
          path: /v1/health/live
          port: http-openai
        initialDelaySeconds: 15
        periodSeconds: 10
        timeoutSeconds: 1
        successThreshold: 1
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /v1/health/ready
          port: http-openai
        initialDelaySeconds: 15
        periodSeconds: 10
        timeoutSeconds: 1
        successThreshold: 1
        failureThreshold: 3
      startupProbe:
        httpGet:
          path: /v1/health/ready
          port: http-openai
        initialDelaySeconds: 360
        periodSeconds: 10
        timeoutSeconds: 1
        successThreshold: 1
        failureThreshold: 120
      env:
        - name: NGC_API_KEY
          valueFrom:
            secretKeyRef:
              name: ngc-api-key-sealed-instruct
              key: api-key
      resources:
        limits:
          nvidia.com/pgpu: "1"
          cpu: "16"
          memory: "48Gi"
      volumeMounts:
        - name: nim-trusted-cache
          mountPath: /opt/nim/.cache
      volumeDevices:
        - name: trusted-storage
          devicePath: /dev/trusted_store
  volumes:
    - name: nim-trusted-cache
      emptyDir:
        sizeLimit: 64Gi
    - name: trusted-storage
      persistentVolumeClaim:
        claimName: trusted-pvc-instruct
EOF
```

只做服务端 dry-run 校验，不实际创建资源：

```bash
kubectl apply --dry-run=server -f "${NIM_TEE_MANIFEST}"
```

如果 dry-run 正常，应该看到 Secret 和 Pod 被报告为 dry-run created 或 configured。这里引用的镜像拉取 Secret 和可信存储 PVC 此时还不需要存在，真正部署前再创建即可。

此时Manifest里还没有 `cc_init_data` 。Checkpoint 5 会通过 `nim-initdata.toml` 和 `genpolicy` 把它加上。

## Checkpoint 4：创建 Pod initdata 注解

这一节为 Guest 内的 [AA 和 CDH](https://confidentialcontainers.org/zh-cn/docs/architecture/design-overview/) 生成配置。生成的 [initdata](https://confidentialcontainers.org/zh-cn/docs/features/initdata/) 会包含 KBS 地址、KBS 证书、用于认证仓库访问的 [registry credential URI](https://confidentialcontainers.org/zh-cn/docs/features/authenticated-registries/)，以及 [image signature policy](https://confidentialcontainers.org/zh-cn/docs/features/signed-images/) URI。

需要注意的是，这份 initdata 虽然通过 Pod 字段由宿主侧提供，但 Guest 不能因为“收到了”就直接信任它。对于 SNP 启动，Kata 会对生成的 initdata 文档做哈希，然后把摘要写入 QEMU 的 SNP `HOST_DATA` 启动值中。SNP 会在 attestation evidence 中提供这个值。Guest 拿到 initdata 后，会验证它的哈希是否与 `HOST_DATA` 一致；证明链路再把校验后的摘要暴露给 KBS。Checkpoint 7 会把这个摘要注入到 KBS 资源策略中。

`KBS_URL` 必须与 checkpoint 2 中使用的地址一致，证书也必须是该地址实际提供的证书。之所以同时把证书嵌进 `aa.toml` 和 `cdh.toml`，是因为 AA 与 CDH 是两个独立的 Guest 组件，它们都会访问 HTTPS KBS 端点：AA 用它做远程证明，CDH 用它拉取资源并解封签名的 sealed secret。

创建 `nim-initdata.toml`：

```bash
INITDATA_FILE="${KBS_WORKDIR}/nim-initdata.toml"

{
  cat <<EOF
version = "0.1.0"
algorithm = "sha256"

[data]
"aa.toml" = """
[token_configs]
[token_configs.kbs]
url = "${KBS_URL}"
cert = '''
EOF
  cat "${KBS_CERT_FILE}"
  cat <<EOF
'''
"""

"cdh.toml" = """
[kbc]
name = "cc_kbc"
url = "${KBS_URL}"
kbs_cert = '''
EOF
  cat "${KBS_CERT_FILE}"
  cat <<'EOF'
'''

[image]
max_concurrent_layer_downloads_per_image = 1
authenticated_registry_credentials_uri = "kbs:///default/credentials/nvcr"
image_security_policy_uri = "kbs:///default/security-policy/nim"
"""
EOF
} > "${INITDATA_FILE}"
```

先不要把这个文件直接写入 Pod。Checkpoint 5 会把它作为 `genpolicy` 的输入，再由 `genpolicy` 把 Kata agent policy 合入同一份 initdata 文档，并把生成后的 `io.katacontainers.config.hypervisor.cc_init_data` 注解字段写回Manifest。

## Checkpoint 5：生成 Kata agent policy

这一节会对 TEE NIM Pod Manifest 运行 `genpolicy`，并把生成的 [Kata agent policy](https://confidentialcontainers.org/zh-cn/docs/features/initdata/#generating-agent-policies-with-genpolicy) 写入 Pod initdata。运行 `genpolicy` 的环境需要能够访问 `nvcr.io`，因为它在生成策略时会读取镜像元数据。

建议把 `genpolicy` 看作可信构建或发布流程的一部分。它的输出决定了 shim 到 agent 接口在信任边界上的行为约束，因此生产环境应在可信环境中生成、审核并发布这份策略，而不是像教程这样在工作节点交互式执行。

`genpolicy` 会按 Docker 风格凭据查找 registry 认证信息。下面的示例会在 `genpolicy` 工作目录中临时生成一份 `nvcr.io` 的 Docker config，并显式让 `genpolicy` 使用它。如果你的可信构建环境已经有合适的 Docker 凭据，也可以直接复用。

先下载与当前 Kata runtime 版本匹配的官方 Kata tools 发布包：

```bash
GENPOLICY_WORKDIR="${KBS_WORKDIR}/genpolicy"
NIM_TEE_MANIFEST="${NIM_TEE_MANIFEST:-${KBS_WORKDIR}/nvidia-nim-llama-3-1-8b-instruct-tee.yaml}"
INITDATA_FILE="${INITDATA_FILE:-${KBS_WORKDIR}/nim-initdata.toml}"
NIM_POLICY_MANIFEST="${GENPOLICY_WORKDIR}/nvidia-nim-llama-3-1-8b-instruct-tee-policy.yaml"
GENPOLICY_INITDATA="${GENPOLICY_WORKDIR}/nim-initdata.toml"
GENPOLICY_DOCKER_CONFIG="${GENPOLICY_WORKDIR}/docker-config"
KATA_RUNTIME_VERSION="$(/opt/kata/bin/kata-runtime --version \
  | awk '/kata-runtime/ {print $3}')"
KATA_TOOLS_ARCH="amd64"
KATA_TOOLS_NAME="kata-tools-static-${KATA_RUNTIME_VERSION}-${KATA_TOOLS_ARCH}"
KATA_TOOLS_TARBALL="${KATA_TOOLS_NAME}.tar.zst"
KATA_RELEASE_BASE="https://github.com/kata-containers/kata-containers/releases/download"
KATA_TOOLS_URL="${KATA_RELEASE_BASE}/${KATA_RUNTIME_VERSION}/${KATA_TOOLS_TARBALL}"
KATA_TOOLS_EXTRACT_DIR="${GENPOLICY_WORKDIR}/kata-tools"
KATA_TOOLS_DIR="${KATA_TOOLS_EXTRACT_DIR}/opt/kata"
GENPOLICY_BIN="${KATA_TOOLS_DIR}/bin/genpolicy"
GENPOLICY_DEFAULTS="${KATA_TOOLS_DIR}/share/defaults/kata-containers"

mkdir -p "${GENPOLICY_WORKDIR}/genpolicy-settings.d" \
  "${GENPOLICY_DOCKER_CONFIG}" \
  "${KATA_TOOLS_EXTRACT_DIR}"

curl -fL \
  -o "${GENPOLICY_WORKDIR}/${KATA_TOOLS_TARBALL}" \
  "${KATA_TOOLS_URL}"

tar --zstd \
  -xf "${GENPOLICY_WORKDIR}/${KATA_TOOLS_TARBALL}" \
  -C "${KATA_TOOLS_EXTRACT_DIR}"

"${GENPOLICY_BIN}" -v \
  -j "${GENPOLICY_DEFAULTS}"
```

准备 registry 凭据，并把与当前 Kata 版本匹配的 `genpolicy` 默认文件复制到本地工作目录：

```bash
: "${NGC_API_KEY:?set NGC_API_KEY to an NGC API key that can pull the NIM image}"
export NGC_API_KEY

GENPOLICY_AUTH="$(printf '$oauthtoken:%s' "${NGC_API_KEY}" | base64 -w0)"
jq -n \
  --arg username '$oauthtoken' \
  --arg password "${NGC_API_KEY}" \
  --arg auth "${GENPOLICY_AUTH}" \
  '{auths: {"nvcr.io": {username: $username, password: $password, auth: $auth}}}' \
  > "${GENPOLICY_DOCKER_CONFIG}/config.json"

cp "${GENPOLICY_DEFAULTS}/rules.rego" \
  "${GENPOLICY_WORKDIR}/rules.rego"

cp "${GENPOLICY_DEFAULTS}/genpolicy-settings.json" \
  "${GENPOLICY_WORKDIR}/genpolicy-settings.json"

cp "${GENPOLICY_DEFAULTS}/drop-in-examples/20-oci-1.3.0-drop-in.json" \
  "${GENPOLICY_WORKDIR}/genpolicy-settings.d/20-oci-1.3.0-drop-in.json"

cp "${NIM_TEE_MANIFEST}" "${NIM_POLICY_MANIFEST}"
cp "${INITDATA_FILE}" "${GENPOLICY_INITDATA}"
```

这里使用 OCI `1.3.0` 的 drop-in，是为了和 GPU/SNP 的 CI 配置对齐。它会调整生成策略中预期的 OCI 版本。如果你的 containerd 栈使用其他 OCI 版本，请选择对应的 drop-in 或直接调整设置值。

可选项：如果你想在工作负载运行期间使用 `kubectl logs` 查看日志，可以在生成的 Kata agent policy 中开启 `ReadStreamRequest`。这对排障很有帮助，但生产环境一般不建议默认开放宿主侧日志流，除非你明确需要这个能力。

```bash
cat > "${GENPOLICY_WORKDIR}/genpolicy-settings.d/99-observation-read-stream.json" <<'EOF'
[
  {
    "op": "replace",
    "path": "/request_defaults/ReadStreamRequest",
    "value": true
  }
]
EOF
```

运行 `genpolicy`。它会原地更新 `NIM_POLICY_MANIFEST`，为其加入 `io.katacontainers.config.hypervisor.cc_init_data` 字段；这个注解包含 checkpoint 4 的 initdata 以及新生成的 `policy.rego`。

```bash
(
  cd "${GENPOLICY_WORKDIR}"

  DOCKER_CONFIG="${GENPOLICY_DOCKER_CONFIG}" \
  RUST_LOG=info \
  "${GENPOLICY_BIN}" \
    -u \
    -y "${NIM_POLICY_MANIFEST}" \
    -p "${GENPOLICY_WORKDIR}/rules.rego" \
    -j "${GENPOLICY_WORKDIR}" \
    --initdata-path="${GENPOLICY_INITDATA}"
)
```

对生成后的清单做 dry-run 校验：

```bash
kubectl apply --dry-run=server -f "${NIM_POLICY_MANIFEST}"
```

检查生成的 `cc_init_data` 注解：

```bash
kubectl create --dry-run=client \
  -f "${NIM_POLICY_MANIFEST}" \
  -o json \
  | jq -r '
      select(.kind == "Pod")
      | .metadata.annotations["io.katacontainers.config.hypervisor.cc_init_data"]
    ' \
  | base64 -d \
  | gzip -d > "${GENPOLICY_WORKDIR}/generated-initdata.toml"

grep -n '^"aa\.toml"[[:space:]]*=' "${GENPOLICY_WORKDIR}/generated-initdata.toml"
grep -n '^"cdh\.toml"[[:space:]]*=' "${GENPOLICY_WORKDIR}/generated-initdata.toml"
grep -n '^"policy\.rego"[[:space:]]*=' "${GENPOLICY_WORKDIR}/generated-initdata.toml"
```

生成后的 initdata 必须同时包含 `aa.toml`、`cdh.toml` 和 `policy.rego`。保留好 `NIM_POLICY_MANIFEST`，checkpoint 6 会复用其中的 `cc_init_data`，而真正部署工作负载时也会直接用到这份清单。

完成这一节后，只需把完整生成的清单留在本地，不要急着部署 NIM 工作负载。应先把参考值、KBS 资源、KBS 策略和可信存储资源都准备好。

## Checkpoint 6：收集 SNP 启动参考值

这一节收集 KBS 策略需要的 SNP 启动输入。只有在你还没有现成且已批准的参考值时，才需要执行本节的手工测量。生产环境中，这些值应根据可信发布制品和预期 VM 启动配置预先计算并审核，这样 KBS 策略可以在工作负载部署前就安装好。

如果你已经有批准过的值，可以直接按后文要求把它们写入 `${KBS_WORKDIR}/nim-snp-reference-values.env`，然后跳到 checkpoint 7，不必再启动测量 Pod。

测量 Pod 可以很轻：不需要运行 NIM 镜像，也不需要真正对外提供服务。它只需要启动出与目标工作负载同类型的机密虚拟机，以便记录启动输入。因此要尽量保持和 TEE NIM Pod 一致的启动相关设置，特别是 runtime class、生成的 `cc_init_data` 注解、GPU 请求以及 CPU/内存规格，因为这些因素都会影响 QEMU 命令行，从而影响 SNP launch measurement。容器镜像本身可以只是一个简单的 sleep 容器。

这里的思路是：先让 Kata 真的把测量 Pod 启起来，再从宿主机侧找到对应的 QEMU 进程，提取 SEV-SNP 启动参数，并交给 `sev-snp-measure` 计算期望的 launch measurement。这样得到的参考值与 Kubernetes+Kata 实际拉起的 VM 形态是对齐的。

生成的 `cc_init_data` 也会影响测量值。Kata 会对该文档计算哈希，并将摘要写到 SNP `HOST_DATA`，后面的 KBS 资源策略会把这个摘要固定下来。

先确认节点处于机密 GPU 模式，并从带策略的清单中提取生成好的 `cc_init_data`：

```bash
kubectl get nodes \
  -L nvidia.com/cc.mode,nvidia.com/cc.mode.state,nvidia.com/cc.ready.state

MEASUREMENT_POD_NAME="nim-snp-measurement"
MEASUREMENT_POD_MANIFEST="${KBS_WORKDIR}/nim-snp-measurement-pod.yaml"
NODE_NAME="$(
  kubectl get nodes \
    -l nvidia.com/cc.mode=on,nvidia.com/cc.mode.state=on,nvidia.com/cc.ready.state=true \
    -o jsonpath='{.items[0].metadata.name}'
)"

CC_INIT_DATA="$(
  kubectl create --dry-run=client \
    -f "${NIM_POLICY_MANIFEST}" \
    -o json \
    | jq -r '
        select(.kind == "Pod")
        | .metadata.annotations["io.katacontainers.config.hypervisor.cc_init_data"]
      '
)"

test -n "${NODE_NAME}"
test -n "${CC_INIT_DATA}"
test "${CC_INIT_DATA}" != "null"
```

创建一个轻量级度量 Pod。它的 CPU、内存和 GPU 限额与 TEE NIM Pod 保持一致，但镜像使用一个简单的 sleep 容器：

```bash
cat > "${MEASUREMENT_POD_MANIFEST}" <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ${MEASUREMENT_POD_NAME}
  annotations:
    io.katacontainers.config.hypervisor.cc_init_data: "${CC_INIT_DATA}"
spec:
  restartPolicy: Never
  runtimeClassName: kata-qemu-nvidia-gpu-snp
  nodeName: ${NODE_NAME}
  containers:
    - name: sleep
      image: quay.io/prometheus/busybox:latest
      imagePullPolicy: IfNotPresent
      command: ["sh", "-c", "sleep 600"]
      resources:
        limits:
          nvidia.com/pgpu: "1"
          cpu: "16"
          memory: "48Gi"
EOF

kubectl delete pod "${MEASUREMENT_POD_NAME}" --ignore-not-found
kubectl apply -f "${MEASUREMENT_POD_MANIFEST}"
```

这里还会用到两个 Kata 本身不提供的宿主机工具：`sev-snp-measure` 用来根据 QEMU 启动参数计算期望的 SNP launch measurement，`snphost` 用来读取宿主机报告的 SNP TCB 值。优先使用发行版包或已批准的 CI 镜像安装它们。若没有现成来源，可使用下面这种常见开发者安装方式。注意：有些发行版会阻止对 externally managed Python 环境执行 `pip install --user`，此时应改用虚拟环境或发行版包。

`snphost` 通过 crates.io 安装，需要 Rust 工具链提供的 `cargo`。`cargo install` 默认会把二进制写到 `${HOME}/.cargo/bin`，因此后面的命令里需要把这个目录加入 `PATH`。

```bash
export PATH="${HOME}/.local/bin:${HOME}/.cargo/bin:${PATH}"

python3 -m pip install --user sev-snp-measure
cargo install --locked snphost

command -v sev-snp-measure
sev-snp-measure --help >/dev/null
command -v snphost
snphost --help >/dev/null
```

找到属于这个 Pod 的 QEMU 进程。这里用 Pod UID 做匹配，避免在同一宿主机上误抓到其他 Kata VM 的 QEMU 进程：

```bash
POD_UID="$(
  kubectl get pod "${MEASUREMENT_POD_NAME}" \
    -o jsonpath='{.metadata.uid}'
)"
POD_UID_UNDERSCORE="${POD_UID//-/_}"

QEMU_PID=""
for _ in $(seq 1 120); do
  QEMU_PID="$(
    for pid in $(pgrep -f 'qemu-system-' || true); do
      if sudo grep -Eq "pod${POD_UID_UNDERSCORE}|${POD_UID}" \
        "/proc/${pid}/cgroup" 2>/dev/null; then
        echo "${pid}"
        break
      fi
    done
  )"
  if test -n "${QEMU_PID}"; then
    break
  fi
  sleep 2
done

test -n "${QEMU_PID}"
echo "${QEMU_PID}"
```

利用这个 QEMU 进程记录准确的启动输入，并计算 SNP launch measurement。脚本会写出 `nim-qemu-launch-inputs.json` 和 `nim-snp-launch-measurement.txt`，同时打印 measurement 和 `SNP_HOST_DATA` 便于快速核对：

```bash
PATH="${PATH}:${HOME}/.local/bin:${HOME}/.cargo/bin" \
QEMU_PID="${QEMU_PID}" \
KBS_WORKDIR="${KBS_WORKDIR}" \
python3 - <<'PY'
import json
import os
from pathlib import Path
import re
import subprocess

pid = os.environ["QEMU_PID"]
cmd = [
    item.decode()
    for item in Path(f"/proc/{pid}/cmdline").read_bytes().split(b"\0")
    if item
]

def value(flag):
    index = cmd.index(flag)
    return cmd[index + 1]

kernel = value("-kernel")
firmware = value("-bios")
append = value("-append").replace(r"\"", '"')
cpu_model = value("-cpu").split(",", 1)[0]
vcpu_count = re.match(r"\d+", value("-smp")).group(0)
initrd = value("-initrd") if "-initrd" in cmd else None
snp_object = next(
    cmd[index + 1]
    for index, item in enumerate(cmd)
    if item == "-object" and cmd[index + 1].startswith("sev-snp-guest")
)
host_data = re.search(r"(?:^|,)host-data=([^,]+)", snp_object).group(1)

out_dir = Path(os.environ["KBS_WORKDIR"])
inputs = {
    "qemu_pid": pid,
    "kernel": kernel,
    "firmware": firmware,
    "initrd": initrd,
    "cpu_model": cpu_model,
    "vcpu_count": int(vcpu_count),
    "append": append,
    "sev_snp_object": snp_object,
    "host_data": host_data,
}
(out_dir / "nim-qemu-launch-inputs.json").write_text(
    json.dumps(inputs, indent=2) + "\n"
)

measure_args = [
    "sev-snp-measure",
    "--mode=snp",
    f"--vcpus={vcpu_count}",
    f"--vcpu-type={cpu_model}",
    "--output-format=hex",
    f"--ovmf={firmware}",
    f"--kernel={kernel}",
    f"--append={append}",
]
if initrd:
    measure_args.append(f"--initrd={initrd}")

measurement = subprocess.check_output(measure_args, text=True).strip()
(out_dir / "nim-snp-launch-measurement.txt").write_text(measurement + "\n")
print(measurement)
print(f"SNP_HOST_DATA={host_data}")
PY
```

然后从同一台宿主机读取 SNP TCB 值。`snphost show tcb` 会从 SNP host device 读取平台状态，后面的 `awk` 再把结果转成 checkpoint 7 直接可用的 shell 变量：

```bash
sudo snphost show tcb | tee "${KBS_WORKDIR}/nim-snp-tcb.txt"

awk '
  /Reported TCB/ { reported = 1; next }
  /Platform TCB/ { reported = 0 }
  reported && /Microcode:/ { print "SNP_MICROCODE=" $2 }
  reported && /SNP:/ { print "SNP_SNP_SVN=" $2 }
  reported && /TEE:/ { print "SNP_TEE_SVN=" $2 }
  reported && /Boot Loader:/ { print "SNP_BOOTLOADER=" $3 }
' "${KBS_WORKDIR}/nim-snp-tcb.txt" > "${KBS_WORKDIR}/nim-snp-tcb.env"

{
  SNP_HOST_DATA_BASE64="$(
    jq -r .host_data "${KBS_WORKDIR}/nim-qemu-launch-inputs.json"
  )"
  SNP_HOST_DATA_HEX="$(
    printf '%s' "${SNP_HOST_DATA_BASE64}" \
      | base64 -d \
      | od -An -tx1 -v \
      | tr -d ' \n'
  )"

  printf 'SNP_LAUNCH_MEASUREMENT=%s\n' "$(
    cat "${KBS_WORKDIR}/nim-snp-launch-measurement.txt"
  )"
  printf 'SNP_HOST_DATA_BASE64=%s\n' "${SNP_HOST_DATA_BASE64}"
  printf 'SNP_HOST_DATA_HEX=%s\n' "${SNP_HOST_DATA_HEX}"
  cat "${KBS_WORKDIR}/nim-snp-tcb.env"
} > "${KBS_WORKDIR}/nim-snp-reference-values.env"

cat "${KBS_WORKDIR}/nim-snp-reference-values.env"
```

再验证在线 Pod 的 `cc_init_data` 注解哈希，是否与 QEMU 启动时写入 SNP `HOST_DATA` 的值一致。这一步能确认测量 Pod 实际收到的 initdata 与证明中看到的是同一份内容：

```bash
kubectl get pod "${MEASUREMENT_POD_NAME}" \
  -o jsonpath='{.metadata.annotations.io\.katacontainers\.config\.hypervisor\.cc_init_data}' \
  | base64 -d \
  | gzip -d > "${KBS_WORKDIR}/live-generated-initdata.toml"

openssl dgst -sha256 -binary "${KBS_WORKDIR}/live-generated-initdata.toml" \
  | base64 -w0
echo

jq -r .host_data "${KBS_WORKDIR}/nim-qemu-launch-inputs.json"
```

这两个值应该一致。

请保留 `nim-snp-reference-values.env`、`nim-qemu-launch-inputs.json`、`nim-snp-launch-measurement.txt` 和 `nim-snp-tcb.txt`，后续审核或复现时都可能用到。收集完成后，删除 Pod：

```bash
kubectl delete pod "${MEASUREMENT_POD_NAME}" --ignore-not-found
```

## Checkpoint 7：写入 KBS 资源、参考值与策略

这一节会把 Guest 侧需要的 [KBS 资源](https://confidentialcontainers.org/zh-cn/docs/attestation/resources/)、checkpoint 6 收集到的 SNP 参考值，以及 [KBS 资源策略](https://confidentialcontainers.org/zh-cn/docs/attestation/resources/) 一并写入 KBS。教程里这些操作直接在工作节点上执行；生产环境应通过正式的 KBS 管理与发布流程完成，并且在工作负载部署前就准备好。

本节会向 KBS 写入以下内容：

- 用于 guest pull 从 `nvcr.io` 拉取镜像的 [认证仓库凭据](https://confidentialcontainers.org/zh-cn/docs/features/authenticated-registries/)，路径为 `default/credentials/nvcr`
- NVIDIA 公钥和容器 [镜像签名策略](https://confidentialcontainers.org/zh-cn/docs/features/signed-images/)
- NIM 运行时使用的明文 API key，路径为 `default/ngc-api-key/instruct`，它会被 checkpoint 3 生成的 sealed Kubernetes Secret 引用
- checkpoint 3 生成的 sealed-secret 验证公钥，供 CDH 在读取 KBS 明文 secret 前先验证 sealed 值签名
- checkpoint 6 收集到的 SNP 启动参考值
- 一条要求 CPU 与 GPU 证明都为 affirming，且 initdata 摘要匹配的 KBS 资源策略

如果你在 checkpoint 2 之后换了 shell，需要先恢复 `KBS_CLIENT`、`KBS_WORKDIR`、`KBS_ADMIN_PRIVATE_KEY_FILE`、`KBS_CERT_FILE` 和 `KBS_URL` 这些变量。

先加载 checkpoint 6 生成的参考值，或者用同样格式加载已经批准的参考值：

```bash
set -a
. "${KBS_WORKDIR}/nim-snp-reference-values.env"
set +a
```

生成供 CDH 在 guest pull 时使用的 NVCR registry auth 文件。它和 NIM 运行时 API key 是两个不同用途的凭据，即使本教程里两者使用了同一个 NGC API key：

```bash
export NGC_API_KEY="<NGC_API_KEY>"

AUTH_VALUE="$(printf '$oauthtoken:%s' "${NGC_API_KEY}" | base64 -w0)"

jq -n --arg auth "${AUTH_VALUE}" '{
  auths: {
    "nvcr.io": {
      auth: $auth
    }
  }
}' > "${KBS_WORKDIR}/nvcr-auth.json"

printf '%s' "${NGC_API_KEY}" > "${KBS_WORKDIR}/ngc-api-key-instruct"
```

创建镜像签名策略，并拉取 NVIDIA 公钥：

```bash
cat > "${KBS_WORKDIR}/nim-image-policy.json" <<'EOF'
{
  "default": [
    {
      "type": "reject"
    }
  ],
  "transports": {
    "docker": {
      "nvcr.io/nim/meta": [
        {
          "type": "sigstoreSigned",
          "keyPath": "kbs:///default/cosign-public-key/nim",
          "signedIdentity": {
            "type": "matchRepository"
          }
        }
      ],
      "nvcr.io/nim/nvidia": [
        {
          "type": "sigstoreSigned",
          "keyPath": "kbs:///default/cosign-public-key/nim",
          "signedIdentity": {
            "type": "matchRepository"
          }
        }
      ]
    }
  }
}
EOF

curl -fsSL \
  -o "${KBS_WORKDIR}/nvidia-cosign.pub" \
  https://api.ngc.nvidia.com/v2/catalog/containers/public-key
```

把资源上传到 KBS。涉及密钥或凭据的上传把输出重定向到 `/dev/null`，避免终端回显编码后的敏感内容：

```bash
"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-resource \
  --path default/credentials/nvcr \
  --resource-file "${KBS_WORKDIR}/nvcr-auth.json" \
  >/dev/null

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-resource \
  --path default/ngc-api-key/instruct \
  --resource-file "${KBS_WORKDIR}/ngc-api-key-instruct" \
  >/dev/null

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-resource \
  --path default/security-policy/nim \
  --resource-file "${KBS_WORKDIR}/nim-image-policy.json"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-resource \
  --path default/cosign-public-key/nim \
  --resource-file "${KBS_WORKDIR}/nvidia-cosign.pub"
```

上传 checkpoint 3 生成的 sealed-secret 公钥。它必须与签名 `ngc-api-key-instruct.sealed` 时使用的私钥匹配，否则 CDH 无法验证 sealed 值：

```bash
"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-resource \
  --path default/signing-key/sealed-secret \
  --resource-file "${KBS_WORKDIR}/signing-key-public.jwk"
```

写入默认 Trustee sample policy 会使用到的 SNP 参考值：

```bash
"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-sample-reference-value \
  snp_launch_measurement "${SNP_LAUNCH_MEASUREMENT}"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-sample-reference-value \
  --as-integer \
  snp_bootloader "${SNP_BOOTLOADER}"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-sample-reference-value \
  --as-integer \
  snp_microcode "${SNP_MICROCODE}"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-sample-reference-value \
  --as-integer \
  snp_snp_svn "${SNP_SNP_SVN}"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-sample-reference-value \
  --as-integer \
  snp_tee_svn "${SNP_TEE_SVN}"
```

`SNP_HOST_DATA_BASE64` 是 initdata 摘要写入 SNP `HOST_DATA` 后的 base64 值。Trustee 在完成 initdata 与 `HOST_DATA` 的校验后，会把相同字节内容以十六进制字符串 `init_data` 暴露在 CPU attestation token 中。这里不会用 `set-sample-reference-value` 去种 `SNP_HOST_DATA_HEX`，而是直接在 KBS 资源策略中检查 `annotated_evidence["init_data"]`，这样可以把整份 initdata 逐字节固定下来。

安装资源策略。它要求 CPU 和 GPU attestation submodule 都为 affirming，同时检查Guest启动时使用的 initdata 摘要是否等于批准值：

```bash
cat > "${KBS_WORKDIR}/nim-kbs-resource-policy.rego" <<EOF
package policy
import rego.v1

default allow = false

cpu0 := input["submods"]["cpu0"]
gpu0 := input["submods"]["gpu0"]
annotated_evidence := cpu0["ear.veraison.annotated-evidence"]
expected_init_data := "${SNP_HOST_DATA_HEX}"

allow if {
    cpu0["ear.status"] == "affirming"
    gpu0["ear.status"] == "affirming"

    annotated_evidence["init_data"] == expected_init_data
}
EOF

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --admin-token-file "${KBS_ADMIN_TOKEN_FILE}" \
  set-resource-policy \
  --policy-file "${KBS_WORKDIR}/nim-kbs-resource-policy.rego"
```

checkpoint 2 的 Compose 部署会把演示状态保存在 `KBS_DATA_DIR`，并把配置与身份材料保存在 `KBS_CONFIG_DIR`。在清理之前不要删除它们。如果 `KBS_DATA_DIR` 被重置，就需要重新执行本节，把资源、参考值和策略重新写回 KBS。生产环境中，KBS 应使用与其信任边界匹配的持久化存储和受保护身份材料。

## Checkpoint 8：准备可信存储资源

这一节会创建 TEE NIM Pod 清单中引用的可信存储资源，但暂时仍不部署 NIM Pod。关于 CoCo 的 [受保护存储](https://confidentialcontainers.org/zh-cn/docs/features/protected-storage/) 模型，以及机密 runtime class 使用的 [confidential emptyDir](https://confidentialcontainers.org/zh-cn/docs/features/protected-storage/confidential-emptydir/) 原语，可参考对应文档。

这里要创建的是 TEE Pod 作为 `/dev/trusted_store` 挂载的可信存储。启用 [guest pull](https://confidentialcontainers.org/zh-cn/docs/getting-started/installation/advanced_configuration/#structured-configuration-kata-containers) 后，CDH 可以把下载的镜像数据写到这个块设备上，而不是塞进 Guest `/run` 文件系统。教程里使用的是 loop-backed 本地 PersistentVolume；生产环境应替换为更适合集群的块存储实现。

示例中使用 `/tmp` 作为 backing file 目录，在很多 CI 系统上它本身就是较大的宿主机 tmpfs。若你的机器空间较小或需要长时间运行，请把 `LOOP_FILE` 换到磁盘文件系统或单独准备的大容量 tmpfs 上。

```bash
NODE_NAME="$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"
LOOP_FILE="/tmp/trusted-image-storage-instruct.img"
STORAGE_SIZE_MIB="57344"

df -h "$(dirname "${LOOP_FILE}")"

# Match Kata's NVIDIA NIM CI setup: remove any stale loop device and fully
# allocate the backing file before the workload starts. A sparse tmpfs file
# created with truncate can charge pages to the pod cgroup as QEMU writes them.
if LOOP_DEVICE="$(sudo losetup -j "${LOOP_FILE}" | awk -F: 'NR == 1 {print $1}')"; \
   test -n "${LOOP_DEVICE}"; then
  sudo losetup --detach "${LOOP_DEVICE}"
fi

rm -f "${LOOP_FILE}"
dd if=/dev/zero of="${LOOP_FILE}" bs=1M count="${STORAGE_SIZE_MIB}" status=progress
LOOP_DEVICE="$(sudo losetup --find --show "${LOOP_FILE}")"

echo "NODE_NAME=${NODE_NAME}"
echo "LOOP_DEVICE=${LOOP_DEVICE}"
```

根据上面的值生成 `trusted-storage-instruct.yaml`：

```bash
cat > trusted-storage-instruct.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: trusted-block-pv-instruct
spec:
  capacity:
    storage: 57344Mi
  volumeMode: Block
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: ${LOOP_DEVICE}
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - ${NODE_NAME}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: trusted-pvc-instruct
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 57344Mi
  volumeMode: Block
  storageClassName: local-storage
EOF
```

应用这些存储对象：

```bash
kubectl apply -f trusted-storage-instruct.yaml

kubectl get pv trusted-block-pv-instruct
kubectl get pvc trusted-pvc-instruct
```

由于这里的 StorageClass 使用 `WaitForFirstConsumer`，所以在 TEE NIM Pod 真正部署前，PVC 保持 `Pending` 是正常现象。

## Checkpoint 9：运行端到端场景

这一节会创建 TEE NIM Pod 所需的 Kubernetes Secret，部署带策略的 TEE NIM 清单，并验证在 KBS 策略生效的情况下 NIM API 仍然可访问。执行前，KBS 资源、SNP 参考值、KBS 资源策略和可信存储都应已准备完成。

```bash
: "${NGC_API_KEY:?set NGC_API_KEY to an NGC API key}"
NIM_POLICY_MANIFEST="${NIM_POLICY_MANIFEST:-${KBS_WORKDIR}/genpolicy/nvidia-nim-llama-3-1-8b-instruct-tee-policy.yaml}"

kubectl create secret docker-registry ngc-secret-instruct \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password="${NGC_API_KEY}" \
  --dry-run=client \
  -o yaml | kubectl apply -f -

kubectl apply -f "${NIM_POLICY_MANIFEST}"

kubectl get secret ngc-api-key-sealed-instruct
kubectl get secret ngc-api-key-sealed-instruct \
  -o jsonpath='{.data.api-key}' | base64 -d | cut -c1-7
echo

kubectl wait --for=condition=Ready \
  --timeout=1000s \
  pod/nvidia-nim-llama-3-1-8b-instruct
```

验证 TEE 工作负载能否通过 NIM API 正常响应：

```bash
POD_IP="$(kubectl get pod nvidia-nim-llama-3-1-8b-instruct -o jsonpath='{.status.podIP}')"

curl -fsS "http://${POD_IP}:8000/v1/health/ready" | jq .
curl -fsS "http://${POD_IP}:8000/v1/models" | jq -r '.data[].id'

MODEL_NAME="$(curl -fsS "http://${POD_IP}:8000/v1/models" | jq -r '.data[0].id')"

curl -fsS "http://${POD_IP}:8000/v1/chat/completions" \
  -H 'Content-Type: application/json' \
  -d "$(jq -n --arg model "${MODEL_NAME}" '{
    model: $model,
    messages: [{role: "user", content: "Reply with exactly: hello from tee nim"}],
    max_tokens: 16,
    temperature: 0
  }')" | jq -r '.choices[0].message.content'
```

如果你在 checkpoint 5 启用了可选的 `ReadStreamRequest` 策略覆盖，那么还可以直接查看容器日志：

```bash
kubectl logs nvidia-nim-llama-3-1-8b-instruct
```

如果 Pod 无法在 KBS 策略下获取资源，优先检查 KBS 日志。参考值缺失或不匹配，通常会表现为 `cpu0` 子模块不是 `affirming`，或者 AS 策略日志中提示某个 reference value identifier 未找到。

如果需要进一步排障，请看后面的 [附录:故障排查](#附录故障排查)。如果你想把节点恢复到一个可重跑教程的干净状态，可直接使用 [附录:故障排查](#附录故障排查)。

## 展望：生产自动化

前面的 checkpoint 展示了如何在单节点上手动跑通机密 NIM 的完整路径。之所以保留手工步骤，是为了让每个信任边界都能被观察到。真正进入生产环境后，这些结果都应该自动化，不应要求操作者在部署时手改清单或执行宿主机命令。

这个流程不存在“一个人包办全部”的合理生产模型。工作负载平台负责把什么 Pod 交给 Kubernetes；发布流程负责生成带策略的制品；集群与 Trustee 团队负责基础信任设施。下面表格把生产职责拆成五类角色，组织内部可以合并角色，但每项责任都应明确归属。

- **集群平台团队或云平台运维**：负责集群 runtime、GPU Operator 集成和 runtime class
- **Trustee 运维方**：在可信环境中运行 Trustee
- **KBS 管理员**：负责 KBS 资源、策略和参考值的写入与维护
- **发布工程或可信发布流水线**：生成 sealed secret、initdata、agent policy 等发布制品
- **模型服务平台或工作负载操作者**：把已批准的工作负载清单提交给 Kubernetes

| 职责项 | 本教程 | 生产角色 | 生产职责 |
| --- | --- | --- | --- |
| 支持 CoCo 的 Kata runtime 与 GPU `RuntimeClass` | 前提条件及 [NVIDIA Confidential Containers 部署指南](https://docs.nvidia.com/datacenter/cloud-native/confidential-containers/latest/confidential-containers-deploy.html) | 集群平台团队或云运维 | 安装并升级 Kata、kata-deploy 或 Helm、GPU Operator，并注册 `kata-qemu-nvidia-gpu-snp` |
| Trustee 服务 | Checkpoint 2：在教程节点上用 Docker Compose 部署 Trustee | Trustee 运维方 | 在可信环境中运行 Trustee，使用持久化存储并审计管理访问 |
| KBS 资源、镜像策略、仓库凭据 | Checkpoint 7：在 worker 上用 `kbs-client` | KBS 管理员 | 通过受批准的管理工具配置资源；变更需版本化与评审 |
| 密封运行时密钥（sealed runtime secrets） | Checkpoint 3、7、9：生成 sealed secret、写入 KBS 资源、再应用到 Kubernetes | 发布工程或可信发布流水线 | 使用集群 CoCo 密封密钥进行密封，仅通过受批准发布路径发布密封对象 |
| TEE Pod 清单与存储依赖 | Checkpoint 3、8：手工编写 manifest 与 PVC | 模型服务平台或工作负载操作者 | 基于固定输入为每个模型/版本产出固定 Pod 模板；向 Kubernetes 提交已批准的 manifest |
| Guest initdata（`aa.toml`、`cdh.toml`） | Checkpoint 4：在 worker 上生成 `nim-initdata.toml` | 发布工程或可信发布流水线 | 产出与已批准 KBS URL 和证书绑定的 initdata |
| Kata agent policy 与 `cc_init_data` | Checkpoint 5：在 worker 上运行 `genpolicy` | 发布工程或可信发布流水线 | 在模型镜像与 Pod 模板固定后，对已批准 Pod manifest 运行 `genpolicy`；评审并与发布物一并发布 |
| SNP 参考值与 KBS 策略 | Checkpoint 6–7：在 worker 上测量启动，再配置 KBS | 发布工程或可信发布流水线；KBS 管理员 | 提前从可信启动产物与预期 VM 配置批准参考值；在部署前安装 KBS 策略 |

## 附录：故障排查

这一节只针对本文这套 NIM 部署路径。建议先看 Kubernetes 状态，再看 KBS 日志。大多数问题通常落在三类：Pod 根本没启动出 sandbox、Guest 启动了但拿不到 KBS 资源，或者 NIM 启动了但迟迟没有 Ready。

先看 Pod 状态和事件：

```bash
kubectl get pod nvidia-nim-llama-3-1-8b-instruct -o wide
kubectl describe pod nvidia-nim-llama-3-1-8b-instruct
kubectl get events \
  --field-selector involvedObject.name=nvidia-nim-llama-3-1-8b-instruct \
  --sort-by=.lastTimestamp
```

对于 Trustee，先看 Compose 服务状态和日志：

```bash
docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  ps

docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  logs --since 30m kbs as rvps
```

下面这些 KBS 日志标记最有参考价值：

- `POST /kbs/v0/auth` 后面紧跟 `POST /kbs/v0/attest`：说明 Guest 已成功访问 KBS 并开始证明
- `Verifier/endorsement check passed. tee=Snp tee_class="cpu"`：说明 SNP 证据校验通过
- `Verifier/endorsement check passed. tee=Nvidia tee_class="gpu"`：说明 GPU 证据校验通过
- `GET /kbs/v0/resource/... 200`：说明资源请求通过了资源策略
- `GET /kbs/v0/resource/default/signing-key/sealed-secret 200`，随后又有 `GET /kbs/v0/resource/default/ngc-api-key/instruct 200`：说明 Guest 通过 CDH 成功解封了运行时 `NGC_API_KEY`
- checkpoint 9 期间如果出现 `No reference value found for the given id: snp_launch_measurement`：通常表示 checkpoint 7 没有把参考值写入当前 KBS，或者写入到了另一个 KBS 实例，或者演示 KBS 存储被清空了
- 如果 `POST /kbs/v0/attest` 成功但后续资源获取被拒绝，通常就是 KBS 资源策略不匹配，需要检查 `cpu0`、`gpu0` 或 initdata 摘要检查哪一项失败
- 如果 `gpu0` 不是 `affirming`，先确认 NVIDIA GPU 固件和驱动栈是否满足当前平台的机密 GPU 要求。GPU 固件过旧会直接导致 GPU 证明失败
- 如果 `credentials/nvcr`、`security-policy/nim`、`cosign-public-key/nim` 这些镜像相关资源都能返回 HTTP 200，但 KBS 从未看到 `signing-key/sealed-secret` 或 `ngc-api-key/instruct` 请求，那么容器大概率拿到的仍是 sealed 版 `NGC_API_KEY`。这时需要重新生成 sealed secret 和 `genpolicy` 输出，并确认 sealed-secret 文件没有多余换行，而且受保护头部格式与 checkpoint 3 一致

有些警告在本教程里并不致命。例如，默认证明策略可能会提示 `snp_smt_enabled` 或 `allowed_vbios_versions` 这类可选参考值缺失。真正需要确认的是：记录下来的 SNP launch measurement 和 TCB 值存在，`cpu0` 与 `gpu0` 最终都变成 `affirming`，并且资源请求返回 HTTP 200。

如果 `kubectl logs nvidia-nim-llama-3-1-8b-instruct` 被拒绝，通常不是 KBS 的问题，而是生成的 Kata agent policy 没允许宿主侧读取日志。若排障期间确实需要日志，请在 checkpoint 5 生成策略前启用可选的 `ReadStreamRequest` 覆盖。生产策略通常不会默认开放这个能力。

NIM 在绑定 8000 端口之前，startup probe 短暂失败并不罕见，尤其是在模型下载、权重加载和 vLLM graph capture 阶段。只有当 startup probe 达到失败阈值或 Pod 进入失败状态时，才需要把它当成问题处理。

如果你要确认当前在线启动是否仍与参考值一致，可以重复 checkpoint 6 的 QEMU 与 `sev-snp-measure` 收集流程。重点关注 kernel append 字符串和 SNP `HOST_DATA`。另外，`/opt/kata/share/defaults/kata-containers/runtimes/qemu-nvidia-gpu-snp/config.d` 里的遗留配置文件也可能改动 QEMU 命令行，从而改变 SNP launch measurement。

KBS、AS 和 RVPS 使用 Rust tracing。Compose 文件会把宿主机环境中的 `RUST_LOG` 传给这些服务。如果你想在启动 Trustee 前提高日志级别，可先导出下面的环境变量，再执行 checkpoint 2 里的 `docker compose up`：

```bash
export RUST_LOG=info,kbs=debug,attestation_service=debug,reference_value_provider_service=debug,policy_engine=debug
```

如果服务已经在运行，可带着这个环境变量重新创建 Compose 服务：

```bash
export RUST_LOG=info,kbs=debug,attestation_service=debug,reference_value_provider_service=debug,policy_engine=debug

docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  up -d --no-build --force-recreate kbs as rvps
```

这会重启 Compose 服务。只要 `KBS_DATA_DIR` 没被删除，教程里用的演示存储在同一台节点上重启容器后仍然会保留 KBS 状态。如果存储目录已重置，就需要重新执行 checkpoint 7 后再测试工作负载。

恢复默认日志级别：

```bash
export RUST_LOG=info

docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  up -d --no-build --force-recreate kbs as rvps
```

生产环境不要长期打开 debug 日志，除非你已经确认日志内容和留存路径不会泄露过多 attestation claim、策略决策细节或资源标识信息。

## 附录：清理

如果你想从 checkpoint 1 重新跑一遍整套教程，可用这一节清理当前节点上的演示资源。它会删除教程工作负载、Trustee Compose 部署、本地 loop 存储以及生成的本地文件。若当前集群是多人共享环境，请先确认这些资源名确实只用于本教程。

```bash
KBS_WORKDIR="${KBS_WORKDIR:-${HOME}/nim-kbs}"
KBS_TRUSTEE_DIR="${KBS_TRUSTEE_DIR:-${KBS_WORKDIR}/trustee}"
GENPOLICY_WORKDIR="${GENPOLICY_WORKDIR:-${KBS_WORKDIR}/genpolicy}"
LOOP_FILE="${LOOP_FILE:-/tmp/trusted-image-storage-instruct.img}"

if test "${KBS_WORKDIR}" = "/" \
   || test "${KBS_TRUSTEE_DIR}" = "/" \
   || test "${GENPOLICY_WORKDIR}" = "/"; then
  echo "Refusing to remove /"
  exit 1
fi

if test -d "${KBS_TRUSTEE_DIR}"; then
  docker compose \
    --project-directory "${KBS_TRUSTEE_DIR}" \
    --project-name nim-trustee \
    down --remove-orphans
fi

kubectl delete pod nvidia-nim-llama-3-1-8b-instruct \
  --ignore-not-found
kubectl delete pod nim-snp-measurement --ignore-not-found

kubectl delete secret \
  ngc-secret-instruct \
  ngc-api-key-instruct \
  ngc-api-key-sealed-instruct \
  --ignore-not-found

kubectl delete pvc trusted-pvc-instruct --ignore-not-found
kubectl delete pv trusted-block-pv-instruct --ignore-not-found
kubectl delete storageclass local-storage --ignore-not-found

if LOOP_DEVICE="$(sudo losetup -j "${LOOP_FILE}" | awk -F: 'NR == 1 {print $1}')" \
   && test -n "${LOOP_DEVICE}"; then
  sudo losetup --detach "${LOOP_DEVICE}"
fi

rm -f "${LOOP_FILE}"

rm -rf "${GENPOLICY_WORKDIR}" || sudo rm -rf "${GENPOLICY_WORKDIR}"
rm -rf "${KBS_WORKDIR}" || sudo rm -rf "${KBS_WORKDIR}"
rm -f trusted-storage-instruct.yaml \
  nvidia-nim-llama-3-1-8b-instruct.yaml \
  nvidia-nim-llama-3-1-8b-instruct-tee.yaml
```

如果你在实验中额外创建过 runtime 配置 drop-in，请在重跑教程前先检查：

```bash
sudo ls -la \
  /opt/kata/share/defaults/kata-containers/runtimes/qemu-nvidia-gpu-snp/config.d
```

只删除你明确为本教程创建的文件。
