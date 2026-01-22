---
title: Init-Data
date: 2025-07-02
description: Use Init-Data to inject dynamic configurations for Pods
weight: 30
categories:
- feature 
tags:
- attestation
- images
- resources
---

Confidential Containers need to access configuration data that cannot be practically embedded in the OS image, such as:

- URIs and certificates required to access Key Broker Services
- **Agent Policy** that is supposed to be enforced by the policy engine
- Configuration regarding image pull, such as proxies and configuration URIs.

In CoCo project this data is referred to as [Init-Data](https://github.com/confidential-containers/trustee/blob/162c620fd9bcd8d6db4bb5b0a5944932a160e89f/kbs/docs/initdata.md). It is specified in TOML format.

Remote attestation ensures the integrity of Init-Data.

## Agent Policy Overview

The **Agent Policy** is one of the most critical components you'll configure via Init-Data. It acts as a security policy engine inside the TEE, controlling what operations the Kata agent can perform.

### Why Agent Policies Matter

Confidential Containers run inside a TEE, but they still interact with the Kubernetes control plane. A malicious or compromised control plane could attempt to:
- Execute arbitrary commands in your container (`kubectl exec`)
- Launch unauthorized container images
- Extract secrets or sensitive data

The agent policy enforces restrictions on these operations **inside the TEE**,
preventing the control plane from compromising your workload's integrity.
More specifically, the agent policy restricts the API between the kata shim
running in the host and the kata agent running in the confidential guest.

### What Agent Policies Control

Agent policies are written in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) and can control:

| API Call | What It Allows | Example Use Case |
|----------|----------------|------------------|
| `CreateContainerRequest` | Which container images can be launched | Allowlist specific image digests |
| `ExecProcessRequest` | Which commands can be executed | Block `kubectl exec` or allow specific commands only |
| `CopyFileRequest` | File operations | Restrict file copying to/from containers |
| `PullImageRequest` | Image pulling operations | Control which registries are accessible |

See the complete [Kata Agent API](https://github.com/kata-containers/kata-containers/blob/main/src/libs/protocols/protos/agent.proto) for all available request types.

### Example: Restrictive Agent Policy

Here's a policy that only allows specific images and blocks all exec operations:

```rego
package agent_policy

# Allow management operations
default CreateSandboxRequest := true
default DestroySandboxRequest := true
default GuestDetailsRequest := true
default StartContainerRequest := true
default StatsContainerRequest := true

# Block exec by default
default ExecProcessRequest := false

# Block container creation by default
default CreateContainerRequest := false

# Only allow specific image digests
CreateContainerRequest if {
    some storage in input.storages
    storage.source == "docker.io/library/nginx@sha256:e56797eab4a5300158cc015296229e13a390f82bfc88803f45b08912fd5e3348"
}

# Optionally allow specific commands
ExecProcessRequest if {
    input_command := concat(" ", input.process.Args)
    input_command == "whoami"
}
```

## Generating Agent Policies with genpolicy

Writing agent policies by hand can be tedious and error-prone for complex workloads. The **genpolicy** tool automatically generates appropriate policies from your Kubernetes manifests.

### Installing genpolicy

The genpolicy tool is included in Kata Containers releases:

1. Download from [Kata Containers releases](https://github.com/kata-containers/kata-containers/releases)
2. Or build from source in the [kata-containers repository](https://github.com/kata-containers/kata-containers/tree/main/src/tools/genpolicy)

### Using genpolicy to Create Policy for Init-Data

**Step 1: Create your Kubernetes manifest**

You can either write a pod manifest manually or generate one using `kubectl`:

```bash
# Option A: Create a pod.yaml manually
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secure
spec:
  runtimeClassName: kata-qemu-tdx
  containers:
  - name: nginx
    image: docker.io/library/nginx@sha256:e56797eab4a5300158cc015296229e13a390f82bfc88803f45b08912fd5e3348
EOF

# Option B: Generate using kubectl --dry-run
kubectl create deployment nginx-secure \
    --image=docker.io/library/nginx@sha256:e56797eab4a5300158cc015296229e13a390f82bfc88803f45b08912fd5e3348 \
    --dry-run=client \
    -o yaml > deployment.yaml
```

**Step 2: Generate the policy**

Run genpolicy on your manifest (works with YAML or JSON):

```bash
genpolicy -y pod.yaml > policy.rego
# or
genpolicy -y deployment.yaml > policy.rego
```

The generated policy will:
- Allow the specific image digests referenced in your manifest
- Set appropriate defaults for container operations
- Include necessary management operations

**Step 3: Review and customize**

The generated policy is a starting point. Review it and customize based on your security requirements.

**Step 4: Include in Init-Data TOML**

Copy the contents of your policy file into your `initdata.toml`:

```toml
version = "0.1.0"
algorithm = "sha384"

[data]
"policy.rego" = '''
<paste contents of policy.rego here>
'''

"aa.toml" = '''
[token_configs.kbs]
url = "http://your-kbs:8080"
'''

"cdh.toml" = '''
[kbc]
name = "cc_kbc"
url = "http://your-kbs:8080"
'''
```

Then follow the steps in [the example below](#example-embedding-init-data-in-a-pod) to embed the Init-Data in your pod.

For more details, see the [genpolicy documentation](https://github.com/kata-containers/kata-containers/tree/main/src/tools/genpolicy).

{{% alert title="Important" color="warning" %}}
Always review generated policies before production use. Genpolicy creates reasonable defaults, but your specific security requirements may need additional customization.
{{% /alert %}}

## Init-Data Format

Now that you understand agent policies, let's see how to package them along with other configuration into Init-Data.

A typical Init-Data TOML file looks like this:

```toml
version = "0.1.0"
algorithm = "sha384"
[data]
"policy.rego" = '''
  <contents of agent-policy>
'''
"aa.toml" = '''
  <contents of Attestation-Agent configuration TOML>
'''

"cdh.toml" = '''
  <contents of Confidential Data Hub configuration TOML>
'''
```

where,

- `version`: The format version of this Init-Data TOML. Currently, **ONLY** version `0.1.0` is supported.
- `algorithm`: The hash algorithm used to calculate the digest of the Init-Data TOML. This is crucial for [attestation](#use-in-attestation). Acceptable values are `sha256`, `sha384` and `sha512`.
- `data`: A dictionary containing concrete configurable files. Supported entries include:
  - `"policy.rego"`: [Kata Agent Policy](../../../blog/2024/policing-a-sandbox.md) file. This helps to restrict the pod to an expected state during runtime..
  - `"aa.toml"`: [Configuration file](https://github.com/confidential-containers/guest-components/blob/main/attestation-agent/attestation-agent/config.example.toml) for the [Attestation Agent](https://github.com/confidential-containers/guest-components/tree/main/attestation-agent), responsible for remote attestation.
  - `"cdh.toml"`: [Configuration file](https://github.com/confidential-containers/guest-components/blob/main/confidential-data-hub/example.config.toml) for the [Confidential Data Hub](https://github.com/confidential-containers/guest-components/tree/main/confidential-data-hub), responsible for in-pod confidential data flow, including image pulling, image decryption, etc.

The detailed Init-Data format definition can be found in the [referenced specification](https://github.com/confidential-containers/trustee/blob/main/kbs/docs/initdata.md#toml-version).

## Example: Embedding Init-Data in a Pod

This section provides an example of creating Init-Data with an agent policy and deploying it to enable remote attestation.

### Step 1: Prepare an Init-Data TOML

Suppose we have a KBS service listening at `http://1.2.3.4:8080`.

The example `initdata.toml` is as follows. Note that we need to set the KBS URL in the following sections:

- `url` of `[token_configs.kbs]` section in `"aa.toml"`, defining the KBS for remote attestation.
- `url` of `[kbc]` section in `"cdh.toml"`, defining the KBS for accessing confidential resources. In this case, we use the same KBS for both attestation and resource provision.

```toml
version = "0.1.0"
algorithm = "sha384"
[data]
"policy.rego" = '''
package agent_policy

# Allow management operations
default CreateSandboxRequest := true
default DestroySandboxRequest := true
default GuestDetailsRequest := true
default StartContainerRequest := true
default StatsContainerRequest := true

# Block exec by default
default ExecProcessRequest := false

# Block container creation by default
default CreateContainerRequest := false

# Only allow specific image digests
CreateContainerRequest if {
    some storage in input.storages
    storage.source == "docker.io/library/nginx@sha256:e56797eab4a5300158cc015296229e13a390f82bfc88803f45b08912fd5e3348"
}

# Optionally allow specific commands
ExecProcessRequest if {
    input_command := concat(" ", input.process.Args)
    input_command == "whoami"
}
'''
"aa.toml" = '''
[token_configs]
[token_configs.kbs]
url = "http://1.2.3.4:8080"
'''

"cdh.toml" = '''
[kbc]
name = "cc_kbc"
url = "http://1.2.3.4:8080"
'''
```

{{% alert title="Tip" color="success" %}}
The policy shown above restricts containers to a specific nginx image digest. For your workloads, use `genpolicy` to automatically generate appropriate policies as shown in the [genpolicy section](#generating-agent-policies-with-genpolicy) above.
{{% /alert %}}

### Step 2: Embed the Init-Data in Pod YAML

We can `gzip` compress and `base64` encode the Init-Data TOML.

```bash
initdata=$(cat initdata.toml | gzip | base64 -w0)
```

Then embed it in `pod.yaml`

```bash
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secure
  annotations:
    io.katacontainers.config.hypervisor.cc_init_data: ${initdata}
spec:
  runtimeClassName: kata-qemu-tdx
  containers:
    - name: nginx
      image: docker.io/library/nginx@sha256:e56797eab4a5300158cc015296229e13a390f82bfc88803f45b08912fd5e3348
      imagePullPolicy: Always
EOF
```

Once deployed, the pod will perform attestation with `http://1.2.3.4:8080` whenever confidential resources need to be accessed.

```bash
kubectl apply -f pod.yaml
```

## Integrity and Attestation

One of the key properties of Init-Data is that its integrity is cryptographically verified through remote attestation. 

When you specify Init-Data:
1. A hash of the Init-Data is calculated using the algorithm specified in the TOML
2. This hash is embedded into the TEE's attestation evidence:
   - **For TDX/SNP**: Written to a configuration register at pod launch (TDX's MR_CONFIG_ID, SNP's HOSTDATA)
   - **For vTPM platforms**: Extended into PCR 8 by the Attestation Agent after launch
3. For TDX/SNP, the Attestation Agent validates that the Init-Data hash matches the launch configuration (if mismatch, the pod fails to start)
4. During remote attestation, the Attestation Service verifies the hash matches the hardware evidence and includes the Init-Data claims in the attestation token
5. KBS resource policies can check these verified Init-Data claims in the token to enforce additional requirements before releasing secrets

This ensures that:
- The agent policy hasn't been tampered with
- KBS/AA configuration is authentic
- The workload is running with the intended security configuration
- Resource policies can make access decisions based on verified configuration

For more details on the integrity mechanisms, see the [Policing a Sandbox](../../../blog/2024/policing-a-sandbox.md#integrity-of-init-data) blog post.

## Additional Use Cases

The dynamic configuration mechanism provided by Init-Data is highly flexible. Here are some more features that leverage Init-Data:

- [Local registries](./local-registries.md) - Configure pods to pull from local container registries
- [Image pull proxy](./image-pull-proxy.md) - Route image pulls through a proxy server
