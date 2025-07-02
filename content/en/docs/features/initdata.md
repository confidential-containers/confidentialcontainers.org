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
- Agent Policy that is supposed to be enforced by the policy engine
- Configuration regarding image pull, such as proxies and configuration URIs.

In CoCo project this data is referred to as [Init-Data](https://github.com/confidential-containers/trustee/blob/162c620fd9bcd8d6db4bb5b0a5944932a160e89f/kbs/docs/initdata.md). It is specified in TOML format.

Remote attestation ensures the integrity of Init-Data.

### Format

A typical Init-Data TOML file looks like this:

```toml
version = "0.1.0"
algorithm = "sha256"
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

### Use-In-Attestation

In this section, we will use Init-Data to enable a Pod to perform remote attestation with a specified Trustee.

#### Step 1: Prepare an Init-Data TOML

Suppose we have a KBS service listening at `http://1.2.3.4:8080`.

The example `initdata.toml` is as follows. Note that we need to set the KBS URL in the following sections:

- `url` of `[token_configs.kbs]` section in `"aa.toml"`, defining the KBS for remote attestation.
- `url` of `[kbc]` section in `"cdh.toml"`, defining the KBS for accessing confidential resources. In this case, we use the same KBS for both attestation and resource provision.

```toml
version = "0.1.0"
algorithm = "sha256"
[data]
"policy.rego" = '''
package agent_policy

default AddARPNeighborsRequest := true
default AddSwapRequest := true
default CloseStdinRequest := true
default CopyFileRequest := true
default CreateContainerRequest := true
default CreateSandboxRequest := true
default DestroySandboxRequest := true
default ExecProcessRequest := true
default GetMetricsRequest := true
default GetOOMEventRequest := true
default GuestDetailsRequest := true
default ListInterfacesRequest := true
default ListRoutesRequest := true
default MemHotplugByProbeRequest := true
default OnlineCPUMemRequest := true
default PauseContainerRequest := true
default PullImageRequest := true
default ReadStreamRequest := true
default RemoveContainerRequest := true
default RemoveStaleVirtiofsShareMountsRequest := true
default ReseedRandomDevRequest := true
default ResumeContainerRequest := true
default SetGuestDateTimeRequest := true
default SetPolicyRequest := true
default SignalProcessRequest := true
default StartContainerRequest := true
default StartTracingRequest := true
default StatsContainerRequest := true
default StopTracingRequest := true
default TtyWinResizeRequest := true
default UpdateContainerRequest := true
default UpdateEphemeralMountsRequest := true
default UpdateInterfaceRequest := true
default UpdateRoutesRequest := true
default WaitProcessRequest := true
default WriteStreamRequest := true
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

#### Step 2: Embed the Init-Data in Pod YAML

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
  name: encrypted-image
  annotations:
    io.containerd.cri.runtime-handler: kata-qemu-tdx
    io.katacontainers.config.hypervisor.kernel_params: ' agent.guest_components_procs=confidential-data-hub'
    io.katacontainers.config.runtime.cc_init_data: ${initdata}
spec:
  runtimeClassName: kata-qemu-tdx
  containers:
    - name: test-container
      image: some-registry.io/image
      imagePullPolicy: Always
      command:
        - sleep
        - "30"
EOF
```

Once deployed, the pod will perform attestation with `http://1.2.3.4:8080` whenever confidential resources need to be accessed.

```bash
kubectl apply -f pod.yaml
```


### More-Use-Cases

The dynamic configuration mechanism provided by Init-Data is highly flexible. Here are some more features based on Init-Data.

- [local registries](./local-registries.md)
- [Image pull proxy](./image-pull-proxy.md)