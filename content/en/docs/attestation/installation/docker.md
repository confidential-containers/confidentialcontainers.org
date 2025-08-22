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
git clone https://github.com/confidential-containers/trustee.git
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

### Uninstall

Stop Trustee.
```bash
docker compose down
```
