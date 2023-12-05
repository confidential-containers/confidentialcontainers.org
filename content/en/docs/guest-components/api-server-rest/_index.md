---
title: API Server Rest
description: Documentation for CoCo Restful API Server
categories:
- docs
tags:
- docs
- guest-components
---

{{% alert title="Warning" color="warning" %}}
TODO: This was copied with few adaptations from here: <https://github.com/confidential-containers/guest-components/tree/main/api-server-rest>
This needs to be tested and verified if the instructions still work and needs a rework.
{{% /alert %}}

CoCo guest components use lightweight ttRPC for internal communication to reduce the memory footprint and dependency. But many internal services also needed by containers like `get_resource`, `get_evidence` and `get_token`, we export these services with restful API, now CoCo containers can easy access these API with http client. Here are some examples, for detail info, please refer [rest API](./openapi/api.json)

```console
$ ./api-server-rest --features=all
Starting API server on 127.0.0.1:8006
API Server listening on http://127.0.0.1:8006
```

```console
$ curl http://127.0.0.1:8006/cdh/resource/default/key/1
12345678901234567890123456xxxx
```

```console
$ curl http://127.0.0.1:8006/aa/evidence\?runtime_data\=xxxx
{"svn":"1","report_data":"eHh4eA=="}
```

```console
$ curl http://127.0.0.1:8006/aa/token\?token_type\=kbs
{"token":"eyJhbGciOiJFi...","tee_keypair":"-----BEGIN... "}
```
