---
title: Trustee in Docker 
description: Installing Trustee on Docker compose
weight: 30
categories:
- attestation
tags:
- trustee
- attestation
- installation
- kubernetes
---

Trustee can be installed using Docker Compose.

### Installation

Clone the Trustee repo.
```bash
git clone https://github.com/confidential-containers/trustee.git && cd trustee
```

Run Trustee.
```bash
docker compose up -d
```

#### Admin Setup (Optional)

Trustee admin APIs are protected. An admin keypair is required to use them.
Trustee in Docker Compose will automatically generate an admin keypair.
The private key, which an admin should provide to the KBS client,
will be located at `kbs/config/private.key`.

You can replace the randomly generated admin keypair with the following commands.
```bash
openssl genpkey -algorithm ed25519 > kbs/config/private.key
openssl pkey -in kbs/config/private.key -pubout -out kbs/config/public.pub
```

#### Debug Mode (Optional)

To enable additional debug information, you can set the `RUST_LOG` environment variable.

First, create a file called `debug.env`.
```bash
RUST_LOG=debug
```

Then, you can run Trustee with an additional argument.
```bash
docker compose --env-file debug.env up
```

### Advanced Setup

Docker Compose mounts Trustee configuration files from the Trustee repository itself.
Specifically, the KBS configuration file is located in `kbs/config/docker-compose/kbs-config.toml`,
the Attestation Service configuration is in `/kbs/config/as-config.json`,
and the RVPS configuration is in `/kbs/config/rvps.json`.

These configuration files are read at Trustee startup. If you edit them, restart Trustee (`docker compose restart`).
The configuration options are described [here](https://github.com/confidential-containers/trustee/blob/main/kbs/docs/config.md) and [here](https://github.com/confidential-containers/trustee/blob/main/attestation-service/docs/config.md).

#### Advanced Settings

Some advanced settings that you may want to enable include:
* **HTTPS** HTTPS can be enabled via the KBS configuration file. HTTPS provides an additional level of security on top of the KBS protocol and should be enabled in production environments.
* **Slim Attestation Token** Guests with many devices can create large attestation tokens. In some cases this will outgrow HTTP header limits. When attesting guests with many devices (such as NVIDIA PPCIE), set `verbose_token` to false in the AS config file.
* **Token Duration** The lifetime/duration of the attestation token can be set via the `duration_min` field of the AS config.

### Uninstall

Stop Trustee.
```bash
docker compose down
```

