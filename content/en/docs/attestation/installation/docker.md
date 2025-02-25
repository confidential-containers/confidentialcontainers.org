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

Setup authentication keys.
```bash
openssl genpkey -algorithm ed25519 > kbs/config/private.key
openssl pkey -in kbs/config/private.key -pubout -out kbs/config/public.pub
```

Run Trustee.
```bash
docker compose up -d
```

### Uninstall

Stop Trustee.
```bash
docker compose down
```
