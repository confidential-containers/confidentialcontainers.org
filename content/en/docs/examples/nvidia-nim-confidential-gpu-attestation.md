---
title: NVIDIA confidential NIM deployment
description: >-
  End-to-end NVIDIA NIM deployment with Kata, Trustee Key Broker
  Service, sealed secrets, and GPU attestation
categories:
- examples
tags:
- gpu
- nvidia
- nim
- kbs
- attestation
---

This example adapts an [NVIDIA NIM](https://docs.nvidia.com/nim/index.html) inference deployment on
Kubernetes to run with Confidential Containers. This particular scenario targets one AMD SEV-SNP
Kubernetes worker node with NVIDIA GPU confidential computing support. The same NIM deployment
pattern can be adapted to Intel TDX nodes, but the reference values and attestation policy must be
generated for TDX rather than SNP. Those TDX-specific steps are out of scope for this exercise.

NVIDIA NIM is a set of inference microservices that package foundation models as containers with
optimized runtimes and HTTP APIs for GPU infrastructure. This example starts with a plain NIM Pod
manifest for the `nvcr.io/nim/meta/llama-3.1-8b-instruct:1.13.1` image, which serves the Meta
Llama 3.1 8B Instruct model through a chat completions API. The optional baseline step runs that
manifest with the non-confidential `kata-qemu-nvidia-gpu` runtime class and queries its health,
model list, and chat completion endpoints on port 8000. The confidential scenario uses the
`kata-qemu-nvidia-gpu-snp` runtime class which moves the Pod into a confidential VM, but the change
alone is not sufficient: A secure deployment also needs Trustee's Key Broker Service (KBS), guest
pull, Attestation Agent (AA) and Confidential Data Hub (CDH) configuration, sealed secrets, image
signature policy, a generated Kata agent policy, trusted storage, and a KBS policy that approves
the expected CPU, GPU, and initdata evidence. The checkpoints below add those pieces one at a time.

{{% alert title="Target Audience and Scope" color="primary" %}}
This is a hands-on single-node tutorial for platform and infrastructure engineers who need to
validate a confidential NIM workload end to end using attestation. It is not a production operations
guide for cluster-wide deployment. Several steps are intentionally manual so that each trust
boundary can be inspected, and the tutorial performs worker-node actions such as creating local
backing storage, running Trustee on the same node as the workload, or collecting SNP launch inputs
from the live QEMU process.

In addition, this tutorial intentionally uses a single operator persona for the full exercise. The
same operator performs platform setup, KBS/resource provisioning, policy generation,
observed-launch collection, and workload validation. This keeps the secure end-to-end flow visible
in a single-node tutorial, but it does not model production responsibility boundaries.
Production considerations are called out at the relevant steps, and
[Outlook: Production Automation](#outlook-production-automation) summarizes how to structure the
same flow for production.
{{% /alert %}}

## Prerequisites

Before starting, prepare:

- a Kubernetes cluster with Confidential Containers, Kata, the NVIDIA GPU Operator, and a GPU
  confidential runtime class such as `kata-qemu-nvidia-gpu-snp`. This setup can be prepared by
  following the [NVIDIA Confidential Containers deployment
  guide](https://docs.nvidia.com/datacenter/cloud-native/confidential-containers/latest/confidential-containers-deploy.html)
- access to the Kubernetes worker node, because this tutorial collects SNP launch inputs from the
  live QEMU process
- an [NGC API key](https://docs.nvidia.com/ngc/latest/ngc-catalog-user-guide.html#generating-a-personal-api-key)
  that can pull the NIM image
- `kubectl`, `curl`, `jq`, `openssl`, `oras`, `skopeo`, `envsubst`, `base64`, `tar`, `zstd`, and
  `cargo` installed on the node
- Docker with the Docker Compose plugin installed on the node
- Python packages `cryptography` and `jwcrypto` for the sealed-secret helper snippet installed on
  the node

{{% alert title="Published Artifacts" color="info" %}}
This tutorial uses released or pre-built artifacts wherever suitable artifacts are available, so you
can validate the deployment flow without cloning component repositories or building components from
source. When a suitable artifact is not yet published, the exception is called out locally and kept
limited to the tutorial flow.
{{% /alert %}}

## Recipe Roadmap

The rest of this document builds the confidential deployment in practical checkpoints. Each
checkpoint should leave the cluster in a state that can be observed and corrected before moving on.

1. **Optional Non-Confidential Baseline**: switch the GPU to
   non-confidential mode, deploy the baseline Pod, show that the NIM API is reachable, remove the
   Pod, then switch the GPU back to confidential mode.
2. **Deploy Trustee and Install kbs-client**: deploy Trustee directly on the tutorial node, select
   matching Trustee artifacts, configure the KBS HTTPS endpoint, and set up the KBS admin helper
   tool. This keeps the demo self-contained, but a production deployment should run Trustee and its
   admin operations from a trusted environment.
3. **Prepare the TEE NIM Manifest and Sealed Secret**: create the sealed runtime secret value, then
   define the SNP GPU Pod manifest that references it, the registry pull Secret, and trusted
   storage.
4. **Create the Pod Initdata Annotation**: configure AA and CDH with the KBS address, KBS
   certificate, registry credential URI, and image signature policy URI.
5. **Generate the Kata Agent Policy**: run `genpolicy` on the TEE Pod
   manifest, combine the policy with the initdata, attach the generated `cc_init_data` annotation to
   the manifest, and keep that generated manifest ready without deploying it. This is normally
   done in a trusted build or release environment.
6. **Collect SNP Launch Reference Values**: when the launch reference values are not already
   available, launch a lightweight measurement Pod with the generated `cc_init_data`, record the SNP
   launch measurement, SNP TCB values, and attested initdata digest, then remove the measurement
   Pod. This observed collection is for the demo; production should approve reference values ahead
   of deployment.
7. **Provision KBS Resources, Reference Values, and Policy**: configure NVIDIA image
   signature verification, NGC registry credentials, the NIM runtime API key, SNP reference values,
   the sealed-secret verification key, and the KBS resource policy that requires affirming CPU and
   GPU evidence plus the approved initdata digest.
8. **Prepare the Trusted Storage Resources**: create the trusted storage dependency referenced by
   the TEE Pod manifest.
9. **Exercise the End-to-End Scenario**: create the Kubernetes-side Secret objects, deploy the
   policy-bearing TEE Pod manifest once, and show that the NIM API is reachable under the KBS
   policy.

## Initial NIM Manifest

Export an NGC API key that can pull the NIM image, then create the initial manifest template. You
will only need to deploy this non-confidential manifest if you exercise the optional baseline check
in checkpoint 1.

The manifest uses the NGC API key in two places: `ngc-secret-instruct` lets Kubernetes pull the
image from `nvcr.io`, and `ngc-api-key-instruct` passes the key into the NIM container at runtime.
This recipe uses the same key for both purposes to keep the example small. In practice, you can use
two different API keys: one for registry access and one for the container's runtime logic.

This file is the base manifest for the optional baseline deployment. It keeps `${NGC_API_KEY}` as a
placeholder; the validation and apply steps below use `envsubst` to substitute the exported value
before passing the manifest to `kubectl`.

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

## Checkpoint 1: Optional Non-Confidential Baseline

Skip this checkpoint if you want to proceed directly to the confidential deployment. It is only a
baseline sanity check for the plain NIM Pod manifest running on Kata Containers. It temporarily
puts the GPU in non-confidential mode, runs the non-TEE Pod, then removes the Pod. The checkpoint
uses node labels to request each mode change, and the NVIDIA GPU Operator orchestrates the GPU mode
switch based on those labels. The last step requests confidential GPU mode again so the remaining
checkpoints continue on the confidential deployment path. For more background on the GPU runtime
classes and confidential GPU mode labels, see the
[NVIDIA GPU examples]({{< relref "nvidia-gpu-examples.md" >}}).

Ensure that the node is in VM passthrough mode and non-confidential GPU mode:

```bash
kubectl label nodes --all \
  nvidia.com/gpu.workload.config=vm-passthrough \
  --overwrite

kubectl label nodes --all nvidia.com/cc.mode=off --overwrite
```

Changing the CC mode may restart GPU Operator operands. Wait for the node label to report that the
transition completed, then wait for the GPU Operator controllers to settle. Waiting for all current
Pods directly can be racy here, because the operator may delete and recreate operand Pods during
the mode transition.

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

Deploy the non-confidential manifest:

```bash
: "${NGC_API_KEY:?set NGC_API_KEY to your NGC API key}"

envsubst '${NGC_API_KEY}' \
  < nvidia-nim-llama-3-1-8b-instruct.yaml \
  | kubectl apply -f -

kubectl wait --for=condition=Ready \
  --timeout=600s \
  pod/nvidia-nim-llama-3-1-8b-instruct
```

For the showcase, query the Pod IP directly from a machine that can reach the cluster Pod network,
such as the single-node cluster host. This baseline does not create a Kubernetes Service. In
production, you would usually expose NIM through a Service, Ingress, Gateway, or another controlled
endpoint.

Check that NIM is ready and list the model name:

```bash
POD_IP="$(kubectl get pod nvidia-nim-llama-3-1-8b-instruct -o jsonpath='{.status.podIP}')"

curl -fsS "http://${POD_IP}:8000/v1/health/ready" | jq .
curl -fsS "http://${POD_IP}:8000/v1/models" | jq -r '.data[].id'
```

Send a minimal chat completion request:

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

After the baseline check passes, remove the non-TEE sample Pod and its Kubernetes Secrets:

```bash
kubectl delete pod nvidia-nim-llama-3-1-8b-instruct --ignore-not-found
kubectl delete secret ngc-secret-instruct ngc-api-key-instruct \
  --ignore-not-found
```

Return the node to confidential GPU mode before continuing:

```bash
kubectl label nodes --all nvidia.com/cc.mode=on --overwrite
```

Repeat the same CC mode stabilization checks shown above, but wait for
`nvidia.com/cc.mode.state` to become `on` instead of `off`. Confirm the node reports `cc.mode=on`
and `cc.mode.state=on` before continuing.

## Checkpoint 2: Deploy Trustee and Install kbs-client

This checkpoint deploys Trustee directly on the tutorial node and installs
[`kbs-client`](../../attestation/client-tool/), the admin tool used later to provision KBS
resources, reference values, and resource policy. KBS is the Trustee service that serves the
guest-facing attestation and resource APIs, as well as the admin API used by `kbs-client`.
Trustee can be deployed with Docker Compose, Helm charts, Kubernetes manifests, or another secure
bootstrap process that matches your environment. This tutorial uses Docker Compose because it
starts KBS, the Attestation Service (AS), and the Reference Value Provider Service (RVPS) together
from the Trustee source tree. In the Compose deployment, KBS, AS, and RVPS run as separate
containers: KBS calls AS to verify evidence, and AS calls RVPS for the reference values used
during verification. The Compose file also runs a one-shot `setup` service that creates demo key
material before the long-running services use it. The steps below call out all configuration and
identity material consumed or created by that deployment, because production deployments may provide
those inputs through separate processes, or replace the demo-generated outputs with material that
has been reviewed, signed, or protected separately.

Running Trustee on the same single-node system as the workload is only for this showcase. It
simplifies networking and keeps the recipe self-contained, but it does not provide the trust
separation that a production deployment needs. In production, deploy Trustee in a separate trusted
environment, use durable storage for KBS state, serve the KBS endpoint with a certificate issued by a
trust chain that guests are configured to trust, and restrict the admin interface carefully. For
alternate deployment methods, see the
[Trustee installation](../../attestation/installation/) documentation.

### Set KBS Deployment Parameters

Start by defining the local working directory, the KBS URL that guest components will use, and the
derived paths used by the Compose deployment. Docker Compose publishes KBS on the worker host.
Because this tutorial runs on a single-node cluster, the guest can use the node IP and the published
host port:

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

Most paths are derived from `KBS_WORKDIR`; normally you only choose `KBS_WORKDIR`, `KBS_PORT`, and
optionally `KBS_CLIENT_ARCH`:

- `KBS_TRUSTEE_DIR` holds the downloaded Trustee source tree.
- `KBS_CONFIG_DIR` is mounted into the Compose services as the demo configuration and identity
  directory.
- `KBS_COMPOSE_CONFIG_DIR` contains the KBS TOML configuration used by the Compose deployment.
- `KBS_DATA_DIR` is the Compose-managed demo state root. RVPS reference values and AS state are
  stored in sibling directories under this tree.
- `KBS_STORAGE_DIR` is the KBS resource and resource-policy repository under `KBS_DATA_DIR`.

The `KBS_URL` value is later written into the pod's initdata document so AA and CDH inside the
guest know where to reach KBS. With this Compose deployment, KBS listens on the host port selected
by `KBS_PORT`; the Kata guest reaches it through the node IP. In production, the KBS endpoint is
usually a stable service address in a separate trusted environment, and guest networking must be
designed so that the guest can reach that endpoint.

In production, use storage that matches the Trustee trust boundary, such as a dedicated persistent
volume or a trusted external backend.

### Select Trustee Deployment Artifacts

The next step selects the Trustee source tree, the Compose service images, and the matching
`kbs-client` artifact. In general, use the Trustee version validated for the CoCo release and
guest components in your cluster. The NVIDIA supported-platforms documentation is intended to be
the source that tells you which Trustee version to use for a given platform. See the
[NVIDIA supported platforms](https://docs.nvidia.com/datacenter/cloud-native/confidential-containers/latest/supported-platforms.html)
page for that version mapping. For this single-node tutorial, the installed Kata release already
carries the Trustee source reference in `/opt/kata/versions.yaml`, so the commands below read that
local file directly.

This works well for the tutorial because the commands run on the same node where Kata is installed.
For production, treat release selection as part of your deployment process instead. If you already
know the Trustee reference to use, set `KBS_TRUSTEE_REF` before running the snippet and skip the
automatic lookup:

```bash
KATA_VERSIONS_FILE="${KATA_VERSIONS_FILE:-/opt/kata/versions.yaml}"
KBS_TRUSTEE_REF="${KBS_TRUSTEE_REF:-}"
KBS_IMAGE_REPO="ghcr.io/confidential-containers/staged-images/kbs-grpc-as"
KBS_AS_IMAGE_REPO="ghcr.io/confidential-containers/staged-images/coco-as-grpc"
KBS_RVPS_IMAGE_REPO="ghcr.io/confidential-containers/staged-images/rvps"
KBS_CLIENT_IMAGE_REPO="ghcr.io/confidential-containers/staged-images/kbs-client"

if test -z "${KBS_TRUSTEE_REF}"; then
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

test -n "${KBS_TRUSTEE_REF}"

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

If you set `KBS_TRUSTEE_REF` manually to a release tag or another source reference, also set
`KBS_COMPOSE_IMAGE_TAG` when your Compose images use a different tag naming scheme. The default
assumption above matches the staged images referenced by the Kata release metadata. If the matching
source tree is not available through the default GitHub archive URL, set `KBS_TRUSTEE_TARBALL_URL`
as well.

### Prepare the Trustee Compose Deployment

Download the matching Trustee source tarball to use the Docker Compose logic from the selected
Trustee version. The tutorial uses the source tree for the Compose file and the Compose
configuration files under `kbs/config`:

```bash
rm -rf "${KBS_TRUSTEE_DIR}"
mkdir -p "${KBS_TRUSTEE_DIR}"

curl -fsSL "${KBS_TRUSTEE_TARBALL_URL}" |
  tar -xz --strip-components=1 -C "${KBS_TRUSTEE_DIR}"
```

The Compose input files used by this checkpoint are:

- `docker-compose.yml`, which starts KBS, AS, RVPS, and a one-time `setup` service that generates
  demo admin and AS token-signing keys.
- `kbs/config/docker-compose/kbs-config.toml`, which configures the KBS endpoint, admin
  authentication, and resource storage.
- `kbs/config/as-config.json`, which configures AS, including how AS reaches RVPS and the NVIDIA
  verifier.
- `kbs/config/rvps.json`, which configures RVPS.

The full files are intentionally not reproduced here. Review the selected source tree, or the same
paths in the Trustee repository, if you want to inspect every default.

The upstream Compose file uses `latest` image tags. Pin the KBS, AS, and RVPS
images to the Trustee reference selected above, and publish KBS on the host port selected by
`KBS_PORT`:

```bash
sed -i \
  -e "s#image: ghcr.io/confidential-containers/staged-images/kbs-grpc-as:latest#image: ${KBS_IMAGE_REPO}:${KBS_COMPOSE_IMAGE_TAG}#" \
  -e "s#image: ghcr.io/confidential-containers/staged-images/coco-as-grpc:latest#image: ${KBS_AS_IMAGE_REPO}:${KBS_COMPOSE_IMAGE_TAG}#" \
  -e "s#image: ghcr.io/confidential-containers/staged-images/rvps:latest#image: ${KBS_RVPS_IMAGE_REPO}:${KBS_COMPOSE_IMAGE_TAG}#" \
  -e "s#\"8080:8080\"#\"${KBS_PORT}:8080\"#" \
  "${KBS_TRUSTEE_DIR}/docker-compose.yml"
```

The Compose setup service generates the demo admin keypair and AS token-signing material when the
services start. It does not create the KBS endpoint certificate, so this checkpoint generates demo
`kbs-https.*` material and wires it into the KBS configuration. The subject alternative name must
match the node IP in `KBS_URL` so that the guest and `kbs-client` can verify the HTTPS endpoint:

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

The demo uses three sets of identity material:

- `kbs-https.key` and `kbs-https.crt` identify the KBS HTTPS endpoint.
- `private.key` and `public.pub` authenticate admin API calls from `kbs-client`. KBS is configured
  with the public key, and `kbs-client` signs admin requests with the private key. The Compose setup
  service creates this demo keypair.
- `ca.key`, `ca-cert.pem`, `token.key`, `token-cert.pem`, and `token-cert-chain.pem` let AS sign
  attestation tokens that KBS can verify. The Compose setup service creates this demo token-signing
  material. In this single-node showcase, KBS and AS are started together, so this material is not
  used to express an independent operational boundary. It becomes important in more advanced
  deployments, for example when KBS and AS are operated separately and KBS must trust tokens issued
  by that AS.

For production, decide who owns each of these identities. For example, the KBS endpoint certificate
should usually come from an organizational CA, admin signing keys should be issued and protected by
an approved identity process, and token-signing private keys may need to be generated or protected
by an HSM or another managed key service.

### Deploy Trustee

Start the Compose deployment. The `--project-directory` option makes the relative volume paths in
`docker-compose.yml` resolve inside the downloaded Trustee tree. The Compose file also contains
`build:` sections for local development, so the commands below pull the pinned images first and use
`--no-build` to keep this tutorial on pre-built artifacts:

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

Verify the Compose services. This Trustee version does not expose a dedicated health endpoint, so
the functional KBS check is done after `kbs-client` is installed:

```bash
docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  ps
```

Keep `KBS_URL`, the admin private key, and the KBS HTTPS certificate; later checkpoints use them to
configure KBS resources and to build pod initdata:

```bash
KBS_ADMIN_PRIVATE_KEY_FILE="${KBS_CONFIG_DIR}/private.key"
KBS_CERT_FILE="${KBS_CONFIG_DIR}/kbs-https.crt"
test -s "${KBS_ADMIN_PRIVATE_KEY_FILE}"
test -s "${KBS_CERT_FILE}"
```

### Install kbs-client

Install `kbs-client` for KBS admin operations. The variables above set `KBS_CLIENT_ARTIFACT` to the
staged `kbs-client` artifact built from the selected Trustee reference. The `oras pull` command
below downloads that artifact and extracts the `kbs-client` binary into `${KBS_WORKDIR}/bin`. This
keeps the client matched to the deployed KBS service, including the admin authentication mode. This
does not change KBS state yet; the actual resource provisioning happens in the next checkpoint.

`kbs-client` is an admin tool used to provision and update KBS resources, reference values, and
resource policy. This tutorial runs the extracted binary from the worker host; production
deployments should run equivalent admin operations from a trusted operator environment.

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

Verify the HTTPS endpoint and admin authentication by writing and then deleting a non-secret probe
resource:

```bash
printf 'checkpoint2-probe\n' > "${KBS_WORKDIR}/checkpoint2-probe.txt"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-resource \
  --path default/checkpoint2/probe \
  --resource-file "${KBS_WORKDIR}/checkpoint2-probe.txt"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  delete-resource \
  --path default/checkpoint2/probe
```

Keep this shell, or save `KBS_CLIENT`, `KBS_WORKDIR`, `KBS_ADMIN_PRIVATE_KEY_FILE`,
`KBS_CERT_FILE`, and `KBS_URL` somewhere private so they can be reused while working through the
next checkpoints. Do not remove `KBS_WORKDIR`; it contains the Trustee source tree, Compose
configuration, demo state, and generated tutorial files. Later `kbs-client` commands authenticate to
KBS with the Compose-generated admin private key, so protect it as an administrator credential.

## Checkpoint 3: Prepare the TEE NIM Manifest and Sealed Secret

This checkpoint creates the signed [sealed secret](../../features/sealed-secrets/) value for the
runtime NGC API key, then renders the TEE NIM Pod manifest without deploying it yet. The manifest will
use the SNP GPU runtime class, a Kubernetes image-pull Secret reference, a Kubernetes Secret that
carries the sealed runtime secret, the [trusted storage](../../features/protected-storage/) PVC, and
the probes from the baseline manifest. The image-pull Secret and PVC are created later, before the
Pod is deployed.

Generate the manifest before deployment so `genpolicy` can use the Pod specification as input. The
generated Kata agent policy and generated `cc_init_data` depend on this manifest, and the
measurement checkpoint will reuse that generated `cc_init_data`.

This workload is configured with trusted image storage. With guest pull, the guest needs a place for
large NIM model data and container image layers. Keeping that data in guest memory-backed
filesystems would require a much larger confidential VM. Putting it on a block device attached to
the confidential VM gives the guest a storage target it can own and use for large artifacts without
relying on the host's normal container filesystem as the runtime storage boundary.

There are two NGC-related Secrets:

- `ngc-secret-instruct` is an image-pull Secret used by the Kubernetes and CRI layer for
  authenticated registry metadata access. In production, minimize the scope of this token and
  consider whether your registry setup can avoid exposing broad credentials to the workload cluster
  control plane. It is created later.
- `ngc-api-key-sealed-instruct` contains only a signed sealed value. Kubernetes sees the sealed
  value; CDH unseals it inside the guest by fetching `kbs:///default/ngc-api-key/instruct` from KBS.
  This Secret is included in the manifest rendered below because `genpolicy` resolves
  `secretKeyRef` values from Secret objects in its YAML input.

Generate a P-256 signing key for sealed secrets. The private JWK stays outside KBS; only the public
JWK is uploaded later. CDH uses the public key resource to verify that the sealed NIM runtime secret
came from the expected operator key before resolving the referenced plaintext secret from KBS.
In production, generate and sign sealed runtime secrets in a trusted release pipeline, protect the
private signing key, and publish only the sealed value through the approved workload release path.

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

Create a signed vault sealed secret that points to the KBS-hosted NIM runtime API key. Under normal
circumstances, use the `secret` CLI from the Confidential Containers project to create this value.
Because this document is intended to avoid requiring readers to build CoCo components from scratch,
and because the `secret` CLI is not published as a standalone artifact at the time of writing, the
next snippet is a temporary compatibility helper that emits the same vault sealed-secret format. Do
not use this helper as production secret tooling; use the supported CoCo tooling from a trusted
release pipeline instead. A vault sealed secret is a JWS-signed JSON document containing the KBS
resource URI and provider metadata. The protected header includes `b64` to match the format
produced by the `secret` CLI tool.

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

Render the Pod manifest:

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

Validate the manifest without creating anything:

```bash
kubectl apply --dry-run=server -f "${NIM_TEE_MANIFEST}"
```

The server dry-run should report the Secret and Pod as created or configured in dry-run mode. The
referenced image-pull Secret and trusted storage PVC do not need to exist yet for this validation;
they are created before the workload is deployed.

The `cc_init_data` annotation is still absent at this point. Checkpoint 5 adds it by running
`genpolicy` with `nim-initdata.toml`.

## Checkpoint 4: Create the Pod Initdata Annotation

This checkpoint will add the
[AA and CDH](../../architecture/design-overview/) configuration used by the guest. The
[initdata](../../features/initdata/) will include the KBS address, the KBS certificate, the
[registry credential URI](../../features/authenticated-registries/) for authenticated registry
access, and the [image signature policy](../../features/signed-images/) URI.

The host still supplies this initdata through a Pod annotation, so the guest must not trust it
merely because it was delivered. For SNP launches, Kata hashes the generated initdata document and
configures QEMU with that digest as the SNP `HOST_DATA` launch value. SNP reports this value in
attestation evidence. The guest verifies that the initdata it received hashes to the reported
`HOST_DATA` value, and the attestation flow exposes the verified digest to KBS. Checkpoint 7 pins
that digest in the KBS resource policy.

The KBS URL must be the same address used in checkpoint 2, and the certificate must be the
certificate that KBS serves for that address. The certificate is included in both `aa.toml` and
`cdh.toml` because AA and CDH are separate guest components that each connect to the HTTPS KBS
endpoint. AA uses its KBS configuration for attestation, while CDH uses its KBS configuration to
fetch resources and unseal signed sealed secrets.

Create `nim-initdata.toml`:

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

Do not add this file directly to the Pod yet. Checkpoint 5 uses it as input to `genpolicy`;
`genpolicy` will add the Kata agent policy to the same initdata document and then write the generated
`io.katacontainers.config.hypervisor.cc_init_data` annotation.

## Checkpoint 5: Generate the Kata Agent Policy

This checkpoint will run `genpolicy` against the TEE NIM Pod manifest and attach the generated
[Kata agent policy](../../features/initdata/#generating-agent-policies-with-genpolicy) to the Pod
initdata. The environment running `genpolicy` needs registry access to `nvcr.io`, because
`genpolicy` reads image metadata while constructing the policy.

Treat `genpolicy` as part of the trusted build or release process. Its output polices the
shim-to-agent interface at the trust boundary, so production deployments should generate, review,
and publish this policy from a trusted environment rather than interactively on a worker node.

`genpolicy` reads registry credentials through Docker-style credential lookup. The example below
creates a temporary Docker config for `nvcr.io` in the `genpolicy` work directory and points
`genpolicy` at it. If the trusted build environment already has suitable Docker credentials, you
can use those instead.

Fetch the official Kata tools release asset that matches the installed Kata runtime.

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

Prepare registry credentials and copy the release-matched `genpolicy` defaults into the local work
directory:

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

The OCI `1.3.0` drop-in matches the GPU/SNP CI setup. It adjusts the expected OCI version in the
generated policy. If your containerd stack uses a different OCI version, choose the matching
drop-in or settings value.

Optional: allow `kubectl logs` during the workload run by enabling `ReadStreamRequest` in
the generated Kata agent policy. This is useful while debugging the showcase, but production policy
should usually leave host-side log streaming disabled unless there is an explicit operational
requirement.

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

Run `genpolicy`. It updates `NIM_POLICY_MANIFEST` in place by adding
`io.katacontainers.config.hypervisor.cc_init_data`; that annotation contains the initdata from
checkpoint 4 plus the generated `policy.rego`.

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

Validate the generated manifest without creating anything:

```bash
kubectl apply --dry-run=server -f "${NIM_POLICY_MANIFEST}"
```

Inspect the generated `cc_init_data` annotation:

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

The generated initdata must contain `aa.toml`, `cdh.toml`, and `policy.rego`. Keep
`NIM_POLICY_MANIFEST`; the measurement checkpoint reuses its generated `cc_init_data`, and the
deployment checkpoint deploys it as the TEE NIM workload.

After this checkpoint, keep the fully generated manifest available, but do not deploy the NIM
workload until the reference values, KBS resources, KBS policy, and trusted storage resources have
been prepared.

## Checkpoint 6: Collect SNP Launch Reference Values

This checkpoint collects the SNP launch inputs needed by the KBS policy. It is only needed in
this manual demo when the launch reference values are not already approved and available. In
production, these values should be computed and approved ahead of time from trusted release
artifacts and the expected VM launch configuration, so the KBS policy can be installed before
the workload is ever deployed.

If you already have approved values, provide them in
`${KBS_WORKDIR}/nim-snp-reference-values.env` using the variable names written below and continue
with checkpoint 7 instead of launching the measurement Pod.

The collection Pod can be lightweight: it does not need to run the NIM image or become
application-ready. It only needs to start the same kind of confidential VM so the launch inputs can
be recorded. Keep the launch-relevant settings aligned with the TEE NIM Pod, especially the
runtime class, generated `cc_init_data` annotation, GPU request, and CPU/memory sizing, because those
can affect the QEMU command line and therefore the SNP launch measurement. The container image can
be a small sleep image.

This checkpoint derives the expected launch measurement from the VM that Kata starts for the
collection Pod. The later steps locate the measurement Pod's QEMU process, read the SEV-SNP launch
inputs from that process, and pass them to `sev-snp-measure`. This ties the recorded reference value
to the VM shape created by Kubernetes and Kata for this workload.

The generated `cc_init_data` matters. Kata hashes that document and configures QEMU with the digest
as the SNP `HOST_DATA` launch value; the KBS resource policy will pin that digest.

First, confirm the node is in confidential GPU mode and extract the generated `cc_init_data` from the
policy-bearing manifest:

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

Create a lightweight measurement Pod. The limits mirror the TEE NIM Pod's CPU, memory, and GPU
requests, but the image is a small sleep container:

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

This checkpoint uses two host-side tools that are not installed by Kata itself:
`sev-snp-measure` computes the expected SNP launch measurement from the QEMU launch inputs, and
`snphost` reads the host's reported SNP TCB values. Install them from your distribution or approved CI
image if available. Otherwise, one common developer setup is shown below. Some distributions block
`pip install --user` for externally managed Python installations; in that case, use a virtual
environment or a distribution package instead of overriding the system policy. The `snphost`
installation command uses crates.io and requires `cargo` from a Rust toolchain. `cargo install`
writes the `snphost` binary to `${HOME}/.cargo/bin`, so add that directory to `PATH` for the
commands that follow.

```bash
export PATH="${HOME}/.local/bin:${HOME}/.cargo/bin:${PATH}"

python3 -m pip install --user sev-snp-measure
cargo install --locked snphost

command -v sev-snp-measure
sev-snp-measure --help >/dev/null
command -v snphost
snphost --help >/dev/null
```

Identify the QEMU process that belongs to the measurement Pod. The Pod UID check avoids recording
the launch inputs for another Kata VM running on the same host:

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

Use that QEMU process to record the exact launch inputs and compute the SNP launch measurement.
The script writes `nim-qemu-launch-inputs.json` and `nim-snp-launch-measurement.txt`, then prints
the launch measurement and `SNP_HOST_DATA` for quick inspection:

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

Record the reported SNP TCB values from the same host. `snphost show tcb` reads the platform state
from the SNP host device, and the `awk` command converts the reported values into shell variables
that checkpoint 7 can pass to KBS:

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

Verify that the live `cc_init_data` annotation hashes to the same value QEMU used as the SNP
`HOST_DATA` launch value. This confirms the initdata attached to the measurement Pod is the same
initdata represented in attestation evidence:

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

The two printed values should match.

Keep `nim-snp-reference-values.env`, `nim-qemu-launch-inputs.json`,
`nim-snp-launch-measurement.txt`, and `nim-snp-tcb.txt` as audit material so the recorded values
can be reviewed or reproduced. Then remove the measurement Pod:

```bash
kubectl delete pod "${MEASUREMENT_POD_NAME}" --ignore-not-found
```

## Checkpoint 7: Provision KBS Resources, Reference Values, and Policy

This checkpoint provisions the guest-facing [KBS resources](../../attestation/resources/), the SNP
reference values from checkpoint 6, and the
[KBS resource policy](../../attestation/policies/#kbs-resource-policies). In this demo these steps
run interactively from the worker; in production they should be handled by approved KBS
administration and release workflows before the workload is deployed.

This checkpoint installs into KBS:

- the [authenticated registry credentials](../../features/authenticated-registries/) for
  guest-pulling from `nvcr.io`, stored as `default/credentials/nvcr`
- NVIDIA's public key and a container [image signature policy](../../features/signed-images/)
- the plaintext NIM runtime API key, stored as `default/ngc-api-key/instruct` and referenced by the
  sealed Kubernetes Secret generated in checkpoint 3
- the sealed-secret verification public key from checkpoint 3, used by CDH to verify the signed
  sealed value before requesting the referenced plaintext KBS resource
- the SNP launch reference values recorded in checkpoint 6
- the KBS resource policy that requires affirming CPU and GPU evidence plus the approved initdata
  digest

If you opened a new shell after checkpoint 2, restore `KBS_CLIENT`, `KBS_WORKDIR`,
`KBS_ADMIN_PRIVATE_KEY_FILE`, `KBS_CERT_FILE`, and `KBS_URL` before running this checkpoint.

Load the reference values recorded in checkpoint 6, or the approved values supplied in the same
format:

```bash
set -a
. "${KBS_WORKDIR}/nim-snp-reference-values.env"
set +a
```

Create the NVCR registry auth file that CDH will use for guest pull. This is separate from the NIM
runtime API key, even though this showcase uses the same NGC API key value for both:

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

Create the image signature policy and fetch NVIDIA's public key:

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

Upload the resources to KBS. The secret-bearing uploads redirect output so the terminal does not
echo base64-encoded credential material:

```bash
"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-resource \
  --path default/credentials/nvcr \
  --resource-file "${KBS_WORKDIR}/nvcr-auth.json" \
  >/dev/null

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-resource \
  --path default/ngc-api-key/instruct \
  --resource-file "${KBS_WORKDIR}/ngc-api-key-instruct" \
  >/dev/null

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-resource \
  --path default/security-policy/nim \
  --resource-file "${KBS_WORKDIR}/nim-image-policy.json"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-resource \
  --path default/cosign-public-key/nim \
  --resource-file "${KBS_WORKDIR}/nvidia-cosign.pub"
```

Upload the sealed-secret public key generated in checkpoint 3. This must match the private key that
signed `ngc-api-key-instruct.sealed`, because CDH will verify the sealed value before resolving the
referenced plaintext secret from KBS.

```bash
"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-resource \
  --path default/signing-key/sealed-secret \
  --resource-file "${KBS_WORKDIR}/signing-key-public.jwk"
```

Seed the SNP reference values used by the default Trustee sample policy:

```bash
"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-sample-reference-value \
  snp_launch_measurement "${SNP_LAUNCH_MEASUREMENT}"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-sample-reference-value \
  --as-integer \
  snp_bootloader "${SNP_BOOTLOADER}"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-sample-reference-value \
  --as-integer \
  snp_microcode "${SNP_MICROCODE}"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-sample-reference-value \
  --as-integer \
  snp_snp_svn "${SNP_SNP_SVN}"

"${KBS_CLIENT}" \
  --cert-file "${KBS_CERT_FILE}" \
  --url "${KBS_URL}" \
  config \
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-sample-reference-value \
  --as-integer \
  snp_tee_svn "${SNP_TEE_SVN}"
```

`SNP_HOST_DATA_BASE64` is the initdata digest used as the SNP `HOST_DATA` launch value. Trustee
exposes the same bytes as the hex string `init_data` in the CPU attestation token after verifying
the supplied initdata against the reported `HOST_DATA` value. This checkpoint does not seed
`SNP_HOST_DATA_HEX` through `set-sample-reference-value`; instead, the KBS resource policy
checks `annotated_evidence["init_data"]` directly. This pins the full initdata document
byte-for-byte.

Install the resource policy. It requires both CPU and GPU attestation submodules to be
affirming and checks that the guest booted with the approved initdata digest:

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
  --auth-private-key "${KBS_ADMIN_PRIVATE_KEY_FILE}" \
  set-resource-policy \
  --policy-file "${KBS_WORKDIR}/nim-kbs-resource-policy.rego"
```

The Compose deployment in checkpoint 2 keeps demo state under `KBS_DATA_DIR` and demo configuration
and identity material under `KBS_CONFIG_DIR`. Keep both directories intact until cleanup; if
`KBS_DATA_DIR` is reset during the tutorial, rerun this checkpoint to repopulate KBS resources,
reference values, and policy. A production KBS deployment should use durable storage and protected
identity material appropriate for its trust boundary.

## Checkpoint 8: Prepare the Trusted Storage Resources

This checkpoint creates the trusted storage resources referenced by the TEE NIM Pod manifest,
without deploying the NIM Pod yet. The CoCo feature documentation describes the broader
[protected storage](../../features/protected-storage/) model and the
[confidential emptyDir](../../features/protected-storage/confidential-emptydir/) primitive used by
confidential runtime classes.

Create the trusted storage resources that the TEE Pod will attach as `/dev/trusted_store`. Later,
when [guest pull](../../getting-started/installation/advanced_configuration/#structured-configuration-kata-containers)
is enabled, CDH can use this device for downloaded image data instead of storing large layers under
the guest's memory-backed `/run` filesystem.
For this showcase, a loop-backed local PersistentVolume is used. In
production, use an appropriate block storage implementation for the cluster.

The example below uses `/tmp`, which is often a large host tmpfs on CI systems. On smaller
or long-running systems, point `LOOP_FILE` at a disk-backed filesystem or a dedicated tmpfs with
enough free space instead.

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

Render `trusted-storage-instruct.yaml` from the values above:

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

Apply the storage objects:

```bash
kubectl apply -f trusted-storage-instruct.yaml

kubectl get pv trusted-block-pv-instruct
kubectl get pvc trusted-pvc-instruct
```

The PVC can remain `Pending` until the TEE NIM Pod is deployed because this StorageClass uses
`WaitForFirstConsumer`.

## Checkpoint 9: Exercise the End-to-End Scenario

This checkpoint creates the Kubernetes Secrets referenced by the TEE NIM Pod, deploys the
policy-bearing TEE NIM manifest once, and shows that the NIM API is reachable under the KBS policy.
At this point, KBS resources, SNP reference values, the KBS resource policy, and trusted storage
should already be in place.

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

Show that the TEE workload is reachable through the NIM API:

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

If you enabled the optional `ReadStreamRequest` policy override in checkpoint 5, you can also
inspect the container logs:

```bash
kubectl logs nvidia-nim-llama-3-1-8b-instruct
```

If the Pod cannot fetch KBS resources under the KBS policy, check the KBS logs first.
Missing or mismatched reference values normally show up as a non-affirming `cpu0` submodule or as
an AS policy warning about the reference value identifier that was not found.

If you need to troubleshoot the run, see the troubleshooting appendix below. If you want to return
the node to a clean tutorial state, use the cleanup appendix.

## Outlook: Production Automation

The checkpoints above demonstrate the full confidential NIM path on one node. They are
intentionally manual so each trust boundary can be inspected. A production deployment, and any
higher-level serving software that schedules workloads through Kubernetes, should automate the same
outcomes without asking operators to adapt manifests or run host-side commands at deploy time.

No single actor owns the entire flow. The model serving platform or workload operator owns what Pod
Kubernetes schedules; the release pipeline owns policy-bearing artifacts; the cluster and Trustee
operators own trust infrastructure. The table uses these five production actors. An organization can
combine roles, but each responsibility should still have an owner:

- **Cluster platform team or cloud operator**: owns the cluster runtime, GPU Operator integration,
  and runtime classes.
- **Trustee operator**: runs Trustee in a trusted environment.
- **KBS administrator**: provisions KBS resources, policies, and reference values.
- **Release engineering or trusted release pipeline**: produces release artifacts such as sealed
  secrets, initdata, and agent policy from pinned inputs.
- **Model serving platform or workload operator**: submits the approved workload manifest to
  Kubernetes.

| Responsibility | Tutorial | Production actor | Production responsibility |
| --- | --- | --- | --- |
| CoCo-capable Kata runtime and GPU `RuntimeClass` | Prerequisites and [NVIDIA Confidential Containers deployment guide](https://docs.nvidia.com/datacenter/cloud-native/confidential-containers/latest/confidential-containers-deploy.html) | Cluster platform team or cloud operator | Install and upgrade Kata, kata-deploy or Helm, GPU Operator, and register `kata-qemu-nvidia-gpu-snp` |
| Trustee services | Checkpoint 2: Trustee deployed with Docker Compose on the tutorial node | Trustee operator | Run Trustee in a trusted environment with durable storage and audited admin access |
| KBS resources, image policy, registry credentials | Checkpoint 7: `kbs-client` on the worker | KBS administrator | Provision resources through approved admin tooling; changes are versioned and reviewed |
| Sealed runtime secrets | Checkpoints 3, 7, and 9: sealed secret generated, backed by KBS resources, then applied to Kubernetes | Release engineering or trusted release pipeline | Seal secrets with the cluster's CoCo sealing key and publish only the sealed object through the approved release path |
| TEE Pod manifest and storage dependency | Checkpoints 3 and 8: hand-authored manifest and PVC | Model serving platform or workload operator | Produce a fixed Pod template per model/version from pinned inputs; submit the approved manifest to Kubernetes |
| Guest initdata (`aa.toml`, `cdh.toml`) | Checkpoint 4: `nim-initdata.toml` on the worker | Release engineering or trusted release pipeline | Produce initdata tied to the approved KBS URL and certificate |
| Kata agent policy and `cc_init_data` | Checkpoint 5: `genpolicy` run on the worker | Release engineering or trusted release pipeline | Run `genpolicy` on the approved Pod manifest when the model image and Pod template are pinned; review and publish the output with the release |
| SNP reference values and KBS policy | Checkpoints 6-7: measurement launch on the worker, then KBS provisioning | Release engineering or trusted release pipeline; KBS administrator | Approve reference values from trusted launch artifacts and expected VM configuration ahead of time; install the KBS policy before deployment |

## Appendix: Troubleshooting

These troubleshooting notes are tailored to this specific NIM deployment scenario.
Start with Kubernetes state, then KBS logs. Most failures in this recipe fall into one of three
buckets: the Pod never starts its sandbox, the guest starts but cannot fetch KBS resources, or NIM
starts but does not become ready.

Check the Pod state and events:

```bash
kubectl get pod nvidia-nim-llama-3-1-8b-instruct -o wide
kubectl describe pod nvidia-nim-llama-3-1-8b-instruct
kubectl get events \
  --field-selector involvedObject.name=nvidia-nim-llama-3-1-8b-instruct \
  --sort-by=.lastTimestamp
```

For Trustee, inspect the Compose services and log tail:

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

Useful KBS log markers:

- `POST /kbs/v0/auth` followed by `POST /kbs/v0/attest` means the guest reached KBS and started
  attestation.
- `Verifier/endorsement check passed. tee=Snp tee_class="cpu"` means SNP evidence was verified.
- `Verifier/endorsement check passed. tee=Nvidia tee_class="gpu"` means GPU evidence was verified.
- `GET /kbs/v0/resource/... 200` means the resource policy allowed a resource fetch.
- `GET /kbs/v0/resource/default/signing-key/sealed-secret 200` followed by `GET
  /kbs/v0/resource/default/ngc-api-key/instruct 200` means the guest unsealed the runtime
  `NGC_API_KEY` environment value through CDH.
- `No reference value found for the given id: snp_launch_measurement` during checkpoint 9 usually
  means checkpoint 7 did not seed the reference values, seeded them into a different KBS instance,
  or they were lost when the demo KBS storage was reset.
- A successful `POST /kbs/v0/attest` followed by denied resource fetches usually points at the KBS
  resource policy. Check whether `cpu0`, `gpu0`, or the initdata digest check is failing.
- If `gpu0` is not `affirming`, confirm that the NVIDIA GPU firmware and driver stack match the
  confidential GPU requirements for your platform. Out-of-date GPU firmware can cause GPU
  attestation to fail even when the KBS resources and SNP reference values are correct.
- If image-related resources such as `credentials/nvcr`, `security-policy/nim`, and
  `cosign-public-key/nim` return HTTP 200 but KBS never sees the `signing-key/sealed-secret` or
  `ngc-api-key/instruct` fetches, the container may still receive the sealed `NGC_API_KEY` value.
  Regenerate the sealed secret and `genpolicy` output, and check that the sealed-secret file has no
  trailing newline and uses the protected header format shown in checkpoint 3.

Some warnings are not fatal in this showcase. For example, the default attestation policies can
warn about optional reference values such as `snp_smt_enabled` or `allowed_vbios_versions`. The
important workload-run checks are that the recorded SNP launch measurement and TCB values are present,
that `cpu0` and `gpu0` become `affirming`, and that the resource fetches return HTTP 200.

If `kubectl logs nvidia-nim-llama-3-1-8b-instruct` is blocked, that is normally the generated Kata
agent policy, not KBS. Enable the optional `ReadStreamRequest` override in checkpoint 5 before
generating the policy if you want workload logs during the workload run. Production policies usually
leave host-side log streaming disabled unless it is explicitly required.

A transient startup probe failure before NIM binds port 8000 is expected during model download,
weight loading, and vLLM graph capture. Treat it as a problem only if the Pod exhausts the startup
probe failure threshold or enters a failed state.

To confirm that the live launch still matches the recorded reference values, repeat the QEMU and
`sev-snp-measure` collection from checkpoint 6. Pay particular attention to the kernel append
string and SNP `HOST_DATA` launch value. Stale files in
`/opt/kata/share/defaults/kata-containers/runtimes/qemu-nvidia-gpu-snp/config.d` can change the QEMU
command line and therefore the SNP launch measurement.

KBS, AS, and RVPS use Rust tracing. The Compose file passes `RUST_LOG` from the
host environment into those services. To increase logging before starting Trustee, export
`RUST_LOG` before running `docker compose up` in checkpoint 2:

```bash
export RUST_LOG=info,kbs=debug,attestation_service=debug,reference_value_provider_service=debug,policy_engine=debug
```

For an already running demo deployment, recreate the Compose services with that environment:

```bash
export RUST_LOG=info,kbs=debug,attestation_service=debug,reference_value_provider_service=debug,policy_engine=debug

docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  up -d --no-build --force-recreate kbs as rvps
```

This restarts the Compose services. With the `KBS_DATA_DIR` demo storage used by this recipe,
KBS state survives a container restart on the same node. If that storage directory is reset, rerun
checkpoint 7 before testing the workload again. To return to the default log level:

```bash
export RUST_LOG=info

docker compose \
  --project-directory "${KBS_TRUSTEE_DIR}" \
  --project-name nim-trustee \
  up -d --no-build --force-recreate kbs as rvps
```

Do not leave debug logging enabled in production unless you have reviewed the log content and
retention path. Debug logs can expose detailed attestation claims, policy decisions, and resource
identifiers.

## Appendix: Cleanup

Use this appendix when you want to rerun the tutorial from checkpoint 1 on the same demo node. It
removes the tutorial workload, Trustee Compose deployment, local loop storage, and generated local
files. Do not run this on a shared cluster unless these names are dedicated to this recipe.

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

If you created additional runtime configuration drop-ins while experimenting, inspect them before
rerunning the tutorial:

```bash
sudo ls -la \
  /opt/kata/share/defaults/kata-containers/runtimes/qemu-nvidia-gpu-snp/config.d
```

Remove only files you intentionally created for this tutorial.
