---
title: KBS Client Tool
description: Simple tool to test or configure Key Broker Service and Attestation Service
weight: 20
categories:
- docs
tags:
- docs
- kbs
---

Trustee can be configured using the KBS Client tool.
Other parts of this documentation may assume that this tool is installed.

The KBS Client can also be used inside of an enclave to retrieve resources
from Trustee, either for testing, or as part of a confidential workload
that does not use Confidential Containers.

When using the KBS Client to retrieve resources, it must be built with an additional feature
which is described below.

Generally, the KBS Client should not be used for retrieving secrets from
inside a confidential container. Instead see, [secret resources](../features/get-resource).

### Install

The KBS Client tool can be installed in two ways.

#### With ORAS

Pull the KBS Client with ORAS.

```bash
oras pull ghcr.io/confidential-containers/staged-images/kbs-client:latest
chmod +x kbs-client
```

This version of the KBS Client does not support getting resources
from inside of an enclave.

#### From source

Clone the Trustee repo.
```bash
git clone https://github.com/confidential-containers/trustee.git
```

Build the client
```
cd kbs
make CLI_FEATURES=sample_only cli
sudo make install cli
```

The `sample_only` feature is used to avoid building hardware attesters into the KBS Client.
to retrieve resources.
If you would like to use the KBS Client inside of an enclave to retrieve secrets,
remove the `sample_only` feature.
This will build the client with all attesters, which will require extra dependencies.

### Usage

Other pages show how the client can be used for specific scenarios.
In general most commands will have the same form.
Here is an example command that sets a resource in the KBS.
```bash
kbs-client --url <url-of-kbs> config \
    --auth-private-key <admin-private-key> set-resource \
    --path <resource-name> --resource-file <path-to-resource-file>
```

#### URL
All `kbs-client` commands must take a `--url` flag.
This should point to the KBS.
Depending on how and where Trustee is deployed, the URL will be different.
For example, if you use the docker compose deployment, the KBS URL
will typically be the IP of your local node with port 8080.

#### Private Key

All `kbs-client config` commands must take an `--auth-private-key` flag.
Configuring the KBS is a privileged operation so the configuration endpoint
must be protected.
The admin private key is usually set before deploying Trustee.
Refer to the installation guide for where your private key is stored,
and point the client to it.
