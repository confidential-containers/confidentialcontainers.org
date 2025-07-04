---
title: Image Pull Proxy
date: 2025-07-02
description: Pull containers from self-hosted registries 
weight: 32
categories:
- feature 
tags:
- local registries
---

In today's cloud-native environments, directly pulling container images from public registries can introduce security and performance challenges. To mitigate these issues, an image pull proxy can act as an intermediary between your environment and external image registries. This document provides a detailed method for configuring a proxy server using [Init-Data](./initdata.md) for CoCo.

### Configuring an Image Pull Proxy with Initdata

It is easy to configure the image pull proxy via Init-Data.

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

[image.image_pull_proxy]

# HTTPS proxy that will be used to pull image
#
# By default this value is not set.
https_proxy = "http://127.0.0.1:5432"

# HTTP proxy that will be used to pull image
#
# By default this value is not set.
http_proxy = "http://127.0.0.1:5432"

# No proxy env that will be used to pull image.
#
# This will ensure that when we access the image registry with specified
# IPs, both `https_proxy` and `http_proxy` will not be used.
#
# If neither `https_proxy` nor `http_proxy` is not set, this field will do nothing.
#
# By default this value is not set.
no_proxy = "192.168.0.1,localhost"
'''
```

> **Note**: a new section `[image.image_pull_proxy]` has been added to `"cdh.toml"`, and the field `https_proxy`, `https_proxy` and `no_proxy` are used to configure proxy-related settings in your environment.

By configuring these settings, you can control the proxy behavior for pulling images, ensuring both security and efficiency in your container deployments. Adjust the proxy addresses and no-proxy settings as needed to fit your network configuration.