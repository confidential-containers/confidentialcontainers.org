---
date: 2024-08-15
title: Policing a Sandbox
linkTitle: Policing a Sandbox
description: >
  How can we provide integrity guarantees for dynamic workloads in the Confidential Containers stack?
author: Magnus Kulke ([@mkulke](https://github.com/mkulke))
---

{{< figure src="/img/sandbox-0.png" alt="The image is a detailed black-and-white technical drawing of a sandbox area. It features a rectangular sandbox filled with sand, with a shovel and bucket inside. To the right of the sandbox, there is a lifeguard chair. Above the sandbox, there is an overhead view of a life preserver ring. In the background, there are architectural plans and measurements for constructing the sandbox area, including various dimensions and structural details." >}}

In a [previous article]({{< ref "building-trust-into-os-images-for-coco.md" >}}) we discussed how we can establish confidence in the integrity of an OS image for a confidential Guest, that is supposed to host a collocated set (Pod) of confidential containers. The topic of this article will cover the integrity of the more dynamic part of a confidential container's lifecycle. We'll refer to this phase as "runtime", even though from the perspective of Container workload it might well be before the actual execution of an entrypoint or command.

{{< figure src="/img/sandbox-1.png" width="65%" alt="The image is a diagram illustrating the components of a sandbox environment, divided into dynamic and static components. It uses color coding to differentiate between various elements: two blue squares labeled ‘Pod Sandbox’ and ‘Container’ represent the dynamic sandbox, while three green squares labeled ‘Base OS,’ ‘Kata Agent,’ and ‘Guest Components’ represent the static components. A legend at the bottom explains the color coding." >}}

A Confidential Containers (CoCo) OS image contains static components like the Kernel and a root filesystem with a container runtime (e.g kata-agent) and auxiliary binaries to support remote attestation. We’ve seen that those can be covered in a comprehensive measurement that will remain stable across different instantiations of the same image, hosting a variety of Confidential Container workloads.

## Why it's hard

The CoCo project decided to use a Kubernetes Pod as its abstraction for confidential containerized workloads. This is a great choice from a user's perspective, since it's a well-known and well-supported paradigm to group resources and a user simply has to specify a specific `runtimeClass` for their Pod to launch it in a confidential TEE (at least that's the premise).

For the implementers of such a solution, this choice comes with a few challenges. The most prominent one is the dynamic nature of a Pod. A Pod in OCI term is a _Sandbox_, in which one or more containers can be imperatively created, deleted, updated via RPC calls to the container runtime. So, instead of a concrete software that acts in reasonably predictable ways we give guarantees about something that is inherently dynamic.

This would be the sequence of RPC calls that are issued to a Kata agent in the Guest VM (for brevity we'll refer to it as _Agent_ in the text below), if we launch a simple Nginx Pod. There are 2 containers being launched, because a Pod includes the implicit `pause` container:

```
create_sandbox
get_guest_details
copy_file
create_container
start_container
wait_process
copy_file
...
copy_file
create_container
stats_container
start_container
stats_container
```

Furthermore, a Pod is a Kubernetes resource which adds a few hard-to-predict dynamic properties, examples would be `SERVICE_*` environment variables or admission controllers that modify a Pod spec before it's launched. The former is maybe tolerable (although it's not hard to come up with a scenario in which the injection of a malicious environment variable would undermine confidentiality), the latter is definitely problematic. If we assume a Pod spec to express the intent of the user to launch a given workload, we can't blindly trust the Kubernetes Control Plane to respect that intent when deploying a CoCo Pod.

## Restricting the Container Environment

In order to preserve the integrity of a Confidential Pod, we need to observe closely what's happening in the Sandbox to ensure nothing unintended will be executed in it that would undermine confidentiality. There are various options to do that:

1. A locked down Kubernetes Control Plane that only allows a specific set of operations on a Pod. It’s tough and implementation-heavy since the Kubernetes API is very expressive and it’s hard to predict all the ways in which a Pod spec can be modified to launch unintended workloads, but there is active research in this area.
  
	 This could be combined with a secure channel between the user and the runtime in the TEE, that allows users to perform certain administrative tasks (e.g. view logs) from which the k8s control plane is locked out.

2. There could be a log of all the changes that are applied to the sandbox. We can record RPC calls and their request payload into a runtime registers of a hardware TEE (e.g. TDX RTMRs or TPM PCRs), that are included in the hardware evidence. This would allow us to replay the sequence of events that led to the current state of the sandbox and verify that it’s in line with the user’s intent, before we release a confidential secret to the workload.

	However not all TEEs provide facilities for such runtime measurements and as we pointed out above: the sequence of RPC calls might be predictable, but the payload is determined by Kubernetes environment that cannot be easily predicted.

3. We can use a combination of the above two approaches. A policy can describe a set of invariants that we expect to hold true for a Pod (e.g. a specific image layer digest) and relax certain dynamic properties that are deemed acceptable (e.g. the `SERVICE_* environment variables) or we can just flat-out reject calls to a problematic RPC endpoint (e.g. exec in container). The policy is enforced by the container runtime in the TEE on every RPC invocation.

	This is elegant as such a policy engine and core policy fragments can be developed alongside the Agent’s API, unburdening the user from understanding the intricacies of the Agent’s API. To be effective an event log as describe in option #2 would not just need to cover the API but also the underlying semantics of this API.

{{< figure src="/img/sandbox-2.png" width="45%" alt="The image is a diagram illustrating the interaction between different components in a containerized environment. It includes two dashed-line squares labeled ‘Container’ at the top, a green and yellow block at the bottom left labeled ‘Kata Agent’ (green) and ‘Policy’ (yellow), and arrows labeled ‘RPC + Payload’ pointing between these elements." >}}

Kata-Containers currently features an implementation of a policy engine using the popular [Rego](https://www.openpolicyagent.org/docs/latest/policy-language) language. Convenience tooling can assist and automate aspects of authoring a policy for a workload. The following would be an example policy (hand-crafted for brevity, real policy bodies would be larger) in which we allow the launch of specific OCI images, the execution of certain commands, Kata management endpoints, but disallow pretty much everything else during runtime:

```rego
"""
package agent_policy

import future.keywords.in
import future.keywords.if
import future.keywords.every

default CopyFileRequest := true
default DestroySandboxRequest := true
default CreateSandboxRequest := true
default GuestDetailsRequest := true
default ReadStreamRequest := true
default RemoveContainerRequest := true
default SignalProcessRequest := true
default StartContainerRequest := true
default StatsContainerRequest := true
default WaitProcessRequest := true

default CreateContainerRequest := false
default ExecProcessRequest := false

CreateContainerRequest if {
	every storage in input.storages {
        some allowed_image in policy_data.allowed_images
        storage.source == allowed_image
    }
}

ExecProcessRequest if {
    input_command = concat(" ", input.process.Args)
	some allowed_command in policy_data.allowed_commands
	input_command == allowed_command
}

policy_data := {
	"allowed_commands": [
		"whoami",
		"false",
		"curl -s http://127.0.0.1:8006/aa/token?token_type=kbs",
	],
	"allowed_images": [
		"pause",
		"docker.io/library/nginx@sha256:e56797eab4a5300158cc015296229e13a390f82bfc88803f45b08912fd5e3348",
	],
}
```

Policies are in many cases dynamic and specific to the workload. Kata ships the [genpolicy tool](https://github.com/kata-containers/kata-containers/tree/365df81d5e98d4cb4f7c2e2e271b9fb503bc4e4c/src/tools/genpolicy) that will generate a reasonable default policy based on a given k8s manifest, which can be further refined by the user. A dynamic policy cannot be bundled in the rootfs, at least not fully, since it needs to be tailored to the workload. This implies we need to provide the Guest VM with the policy at launch time, in a way that allows us to trust the policy to be genuine and unaltered. In the next section we'll discuss how we can achieve this.

## Init-Data

Confidential Containers need to access configuration data that for practical reasons cannot be baked into the OS image. This data includes URIs and certificates required to access Attestation and Key Broker Services, as well as the policy that is supposed to be enforced by the policy engine. This data is not secret, but maintaining its integrity is crucial for the confidentiality of the workload.

In the CoCo project this data is referred to as [Init-Data](https://github.com/confidential-containers/trustee/blob/162c620fd9bcd8d6db4bb5b0a5944932a160e89f/kbs/docs/initdata.md). Init-data is specified as file/content dictionary in the TOML language, optimized for easy authoring and human readability. Below is a (shortened) example of a typical Init-Data block, containing some pieces of metadata, configuration for CoCo guest components and a policy in Rego language:

```toml
algorithm = "sha256"
version = "0.1.0"

[data]
"aa.toml" = '''
[token_configs]
[token_configs.kbs]
url = 'http://my-as:8080'
cert = """
-----BEGIN CERTIFICATE-----
MIIDEjCCAfqgAwIBAgIUZYcKIJD3QB/LG0FnacDyR1KhoikwDQYJKoZIhvcNAQEL
...
4La0LJGguzEN7y9P59TS4b3E9xFyTg==
-----END CERTIFICATE-----
"""
'''

"cdh.toml"  = '''
socket = 'unix:///run/confidential-containers/cdh.sock'
credentials = []
...
'''

"policy.rego" = '''
package agent_policy
...
'''

```

A user is supposed to specify Init-Data to a Confidential Guest in the form of a `gzip` compressed and base64-encoded string in a specific Pod annotation. The Kata Containers runtime will then pass this data on to the Agent in the Guest VM, which will decode the Init-Data and use it to configure the runtime environment of the workload. Crucially, since Init-Data is not trusted at launch we need a way to establish that the policy has not been tampered with in the process.

## Integrity of Init-Data

The Init-Data body that was illustrated above contains a metadata header which specifies a hash algorithm that is supposed to be used to verify the integrity of the Init-Data. Establishing trust in provided Init-Data is not completely trivial.

Let’s start with a naive approach anyway: Upon retrieval and before applying Init-Data in the guest we can calculate a hash of that Init-Data body and stash the measurement away somewhere in encrypted and integrity protected memory. Later we could append it to the TEE’s hardware evidence as an additional fact about the environment. An Attestation-Service would take that additional fact into account and refuse to release a secret to a confidential workload if e.g. a too permissive policy was applied.

### Misdemeanor in the Sandbox

We have to take a step back and look at the bigger picture to understand why this is problematic. In CoCo we are operating a sandbox, i.e. a rather liberal playground for all sorts of containers. This is by design, we want to allow users to migrate existing containerized workloads with as little friction as possible into a TEE. Now we have to assume that some of the provisioned workloads might be malicious and attempting to access secrets they should not have access to. Confidential Computing is also an effort in declaring explicit boundaries.

There are pretty strong claims that VM-based Confidential Computing is secure, because it builds on the proven isolation properties of hardware-based Virtual Machines. Those have been battle-tested in (hostile) multi-tenant environments for decades and the confidentiality boundary between a Host and Confidential Guest VM is defined along those lines.

Now, Kata Containers _do_ provide an isolation mechanism. There is a jail for containers that employs all sorts of Linux technologies (seccomp, Namespaces, Cgroups, …) to prevent a container from breaking out of its confinement. However, containing containers is a hard problem and regularly new ways of container’s escaping their jail are discovered and exploited (adding VM-based isolation to Containers is one of the defining features for Kata Containers, after all).

### Circling back to the Init-Data Measurement

The measurement is a prerequisite for accessing a confidential secret. If we keep such a record in the memory of a CoCo management process within the Guest, this would have implications for the Trust Model: A Hardware Root-of-Trust module is indispensable for Confidential Computing. A key property of that module is the strong isolation from the Guest OS. Through clearly defined interfaces it can record measurements of the guest’s software stack. Those measurements are either static or extend-only. A process in the guest VM cannot alter them freely.

{{< figure src="/img/sandbox-3.png" width="60%" alt="The image is a diagram illustrating the interaction between different components in a virtualized environment. It uses color coding to differentiate between elements: yellow for ‘Policy,’ green for ‘Guest OS,’ pink for ‘Hardware Root of Trust,’ and blue for ‘VM Host.’ The diagram shows communication flows with dashed arrows, indicating how the ‘Policy’ within the ‘Guest OS’ interacts with the ‘Hardware Root of Trust’ through the ‘VM Host." >}}

A measurement record in the guest VM's software stack is not able to provide similar isolation. A process in the guest, like a malicious container, would be able to tamper with such a record and deceive a Relying Party in a Remote Attestation in order to get access to restricted secrets. A user would not only have to trust the CoCo stack to perform the correct measurements before launching a container, but they would also have to trust this stack to not be vulnerable to sandbox escapes. This is a pretty big ask.

Hence a pure software approach to establish trust in Init-Data is not desirable. We want to move the trust boundary back to the TEE and link Init-Data measurements to the TEE’s hardware evidence. There are generally 2 options to establish such a link, which one of those is chosen depends on the capabilities of the TEE:

### Host-Data

Host-Data is a field in a TEE’s evidence that is passed into a confidential Guest from its Host verbatim. It’s not secret, but its integrity is guaranteed, as its part of the TEE-signed evidence body. We are generalising the term Host-Data from SEV-SNP here, a similar concept exists in other TEEs with different names. Host-Data can hold a limited amount of bytes, typically in the 32 - 64 byte range. This is enough to hold a hash of the Init-Data, calculated at the launch of the Guest. This hash can be used to verify the integrity of the Init-Data in the guest, by comparing the measurement (hash) of the Init-Data in the guest with the host-provided hash in the Host-Data field. If the hashes match, the Init-Data is considered to be intact.

Example: Producing a SHA256 digest of the Init-Data file

```bash
openssl dgst -sha256 --binary init-data.toml | xxd -p -c32
bdc9a7390bb371258fb7fb8be5a8de5ced6a07dd077d1ce04ec26e06eaf68f60
```

### Runtime Measurements

Instead of seeding the Init-Data hash into a Host-Data field at launch, we can also extend the TEE evidence with a runtime measurement of the Init-Data directly, if the TEE allows for it. This measurement is then a part of the TEE’s evidence and can be verified as part of the TEE’s remote attestation process.

Example: Extending an empty SHA256 runtime measurement register with the digest of an Init-Data file 

```bash
dd if=/dev/zero of=zeroes bs=32 count=1
openssl dgst -sha256 --binary init-data.toml > init-data.digest
openssl dgst -sha256 --binary <(cat zeroes init-data.digest) | xxd -p -c32
7aaf19294adabd752bf095e1f076baed85d4b088fa990cb575ad0f3e0569f292
```

## Glueing Things Together

Finally, in practice a workflow would look like the steps depicted below. Note that the concrete implementation of the individual steps might vary in future revisions of CoCo (as of this writing v0.10.0 has just been released), so this is not to be taken as a reference but merely to illustrate the concept. There are practical considerations, like limitations in the size of a Pod annotation, or how Init-Data can be provisioned into a guest that might alter details of the workflow in the future.

### Creating a Manifest

kubectl's `--dry-run` option can be used to produce a JSON manifest for a Pod deployment, using the allow-listed image from the policy example above. We are using `jq` to specify a CoCo runtime class:

```bash
kubectl create deployment \
	--image="docker.io/library/nginx@sha256:e56797eab4a5300158cc015296229e13a390f82bfc88803f45b08912fd5e3348" \
	nginx-cc \
	--dry-run=client \
	-o json \
	| jq '.spec.template.spec.runtimeClassName = "kata-cc"' \
	> nginx-cc.json
```

An Init-Data file is authored, then encoded in base64 and added to the Pod annotation before the deployment is triggered:

```bash
vim init-data.toml
INIT_DATA_B64="$(cat "init-data.toml" | gzip | base64 -w0)"
cat nginx-cc.yaml | jq \
	--arg initdata "$INIT_DATA_B64" \
	'.spec.template.metadata.annotations = { "io.katacontainers.config.hypervisor.cc_init_data": $initdata }' \
	| kubectl apply -f -
```

### Testing the Policy

If the Pod came up successfully, it passed the initial policy check for the image already.

```bash
kubectl get pod
NAME                         READY   STATUS        RESTARTS   AGE
nginx-cc-694cc48b65-lklj7    1/1     Running       0          83s
```

According to the policy only certain commands are allowed to be executed in the container. Executing `whoami` should be fine, while `ls` should be rejected:

```bash
kubectl exec -it deploy/nginx-cc -- whoami
root
```

```bash
kubectl exec -it deploy/nginx-cc -- ls
error: Internal error occurred: error executing command in container: failed to
exec in container: failed to start exec "e2d8bad68b64d6918e6bda08a43f457196b5f30d6616baa94a0be0f443238980": cannot enter container 914c589fe74d1fcac834d0dcfa3b6a45562996661278b4a8de5511366d6a4609, with err rpc error: code = PermissionDenied desc = "ExecProcessRequest is blocked by policy: ": unknown
```

In our example we tie the Init-Data measurement to the TEE evidence using a Runtime Measurement into PCR8 of a vTPM. Assuming a 0-initalized SHA256 register, we can calculate the expected value by extend the zeroes with the SHA256 digest of the Init-Data file:

```bash
dd if=/dev/zero of=zeroes bs=32 count=1
openssl dgst -sha256 --binary init-data.toml > init-data.digest
openssl dgst -sha256 --binary <(cat zeroes init-data.digest) | xxd -p -c32
765156eda5fe806552610f2b6e828509a8b898ad014c76ad8600261eb7c5e63f
```

As part of the policy we also allow-listed a specific command that can request a KBS token using an endpoint that is exposed to a container by a [specific Guest Component](https://github.com/confidential-containers/guest-components/tree/41ad96d4b2e5e9dc205c6d41f7b550629cea677f/api-server-rest). Note: This is not something a user would want to typically enable, since this token is used to retrieve confidential secrets and we would not want it to leak outside the Guest. We are using it here to illustrate that we _could_ retrieve a secret in the container, since we passed remote attestation including the verification of the Init-Data digest.


```bash
kubectl exec deploy/nginx-cc -- curl -s http://127.0.0.1:8006/aa/token\?token_type=kbs | jq -c 'keys'
["tee_keypair","token"]
```

Since this has been successful we can inspect the logs of the Attestation Service (bundled into a KBS here) to confirm it has been considered in the appraisal. The first text block shows the claims from the (successfully verified) TEE evidence, the second block is displaying the acceptable reference values for a PCR8 measurement:

```bash
kubectl logs deploy/kbs -n coco-tenant | grep -C 2 765156eda5fe806552610f2b6e828509a8b898ad014c76ad8600261eb7c5e63f
...
        "aztdxvtpm.tpm.pcr06": String("65f0a56c41416fa82d573df151746dc1d6af7bd8d4a503b2ab07664305d01e59"),
        "aztdxvtpm.tpm.pcr07": String("124daf47b4d67179a77dc3c1bcca198ae1ee1d094a2a879974842e44ab98bb06"),
        "aztdxvtpm.tpm.pcr08": String("765156eda5fe806552610f2b6e828509a8b898ad014c76ad8600261eb7c5e63f"),
        "aztdxvtpm.tpm.pcr09": String("1b094d2504eb2f06edcc94e1ffad4e141a3cd5024b885f32179d1b3680f8d88a"),
        "aztdxvtpm.tpm.pcr10": String("bb5dfdf978af8a473dc692f98ddfd6c6bb329caaa5447ac0b3bf46ef68803b17"),
--
        "aztdxvtpm.tpm.pcr08": [
            "7aaf19294adabd752bf095e1f076baed85d4b088fa990cb575ad0f3e0569f292",
            "765156eda5fe806552610f2b6e828509a8b898ad014c76ad8600261eb7c5e63f",
        ],
		"aztdxvtpm.tpm.pcr10": [],
```

### Size Limitations

In practice there are limitations with regards to the size of init-data bodies. Especially the policy sections of such a document can reach considerable size for complex pods and thus exceed the limitations that currently exist for annotation values in a Kubernetes Pod. As of today various options are being discussed to work with this limitation, ranging from simple text-compression to more elaborate schemes.

## Conclusion

In this article we discussed the challenges of ensuring the integrity of a Confidential Container workload at runtime. We’ve seen that the dynamic nature of a Pod and the Kubernetes Control Plane make it hard to predict exactly what will be executed in a TEE. We’ve discussed how a policy engine can be used to enforce invariants on such a dynamic workload and how Init-Data can be used to provision a policy into a Confidential Guest VM. Finally, we’ve seen how the integrity of Init-Data can be established by linking it to the TEE’s hardware evidence.

_Thanks to Fabiano Fidêncio and Pradipta Banerjee for proof-reading and ideas for improvements!_
