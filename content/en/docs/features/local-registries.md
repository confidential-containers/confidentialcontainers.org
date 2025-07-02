---
title: Local Registries
date: 2025-07-02
description: Pull containers from self-hosted registries 
categories:
- feature 
tags:
- local registries
---

In certain scenarios, there arises a need to pull container images directly from local registries, rather than relying on public repositories. This requirement may stem from the desire for reduced latency, enhanced security, or network isolation.

Local registries can be categorized into two types based on their security protocols: HTTPS-secured registries and HTTP-accessible registries.

### HTTPS-Secured Registries

For registries protected by HTTPS, communication security is paramount to ensure that image transmission remains confidential and is received intact. Registry certificate configuration needs to be configured via [Init-Data](./initdata.md) for secure communication.

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

[image]
# To support registries with self signed certs. This config item
# is used to add extra trusted root certifications. The certificates
# must be encoded by PEM.
#
# By default this value is not set.
extra_root_certificates = ["""
-----BEGIN CERTIFICATE-----
MIIFTDCCAvugAwIBAgIBADBGBgkqhkiG9w0BAQowOaAPMA0GCWCGSAFlAwQCAgUA
oRwwGgYJKoZIhvcNAQEIMA0GCWCGSAFlAwQCAgUAogMCATCjAwIBATB7MRQwEgYD
VQQLDAtFbmdpbmVlcmluZzELMAkGA1UEBhMCVVMxFDASBgNVBAcMC1NhbnRhIENs
YXJhMQswCQYDVQQIDAJDQTEfMB0GA1UECgwWQWR2YW5jZWQgTWljcm8gRGV2aWNl
...
-----END CERTIFICATE-----
"""]
'''
```

> **Note:** a new section `[image]` has been added to `"cdh.toml"`, and the field `extra_root_certificates` includes the certificate chain of the local registry.

### HTTP-Accessible Registries

TODO