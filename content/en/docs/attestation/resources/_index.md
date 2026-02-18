---
title: Resources
description: Managing secret resources with Trustee 
weight: 30
categories:
- attestation
tags:
- trustee
- attestation
- secrets
- resources
---

Trustee, and the KBS in particular, are built around providing secrets to workloads.
These secrets are fulfilled by secret backend plugins, the most common of which
is the resource plugin.
In practice the terms secret and resource are used interchangeably.

There are several ways to configure secret resources.

## Identifying Resources

Confidential Containers and Trustee use the Resource URI scheme to identify resources.
A full Resource URI has the form `kbs://<kbs_host>:<kbs_port>/<repository>/<type>/<tag>`,
but in practice the KBS Host and KBS Port are ignored to avoid coupling a resource
to a particular IP and port.

Instead, resources are typically expressed as
```
kbs:///<repository>/<type>/<tag>`
```

Often resources are referred to just as `<repository>/<type>/<tag>`.
This is what the KBS Client refers to as the resource path.

## Providing Resources to Trustee

By default, Trustee supports a few different ways of provisioning resources.

### KBS Client

You can use the KBS Client to provision resources.
```bash
kbs-client --url <kbs-url> config \
    --auth-private-key <admin-private-key> set-resource \
    --path <resource-path> --resource-file <path-to-resource-file>
```

For more details about using the KBS Client, refer to the KBS Client [page](../kbs-client)

### Filesystem Backend

By default Trustee will store resources on the filesystem.
You can provision resources by placing them into the appropriate directory. 
When using Kubernetes, you can inject a secret like this.

```bash
cat "$KEY_FILE" | kubectl exec -i deploy/trustee-deployment -n trustee-operator-system -- tee "/opt/confidential-containers/kbs/repository/${KEY_PATH}" > /dev/null
```

This approach can be extended to integrate the resource backend with other Kubernetes components.

### Operator

If you are using the Trustee operator, you can specify Kubernetes secrets that will be propagated to the resource backend.

A secret can be added like this
```bash
kubectl create secret generic kbsres1 --from-literal key1=res1val1 --from-literal key2=res1val2 -n kbs-operator-system
```

## Advanced configurations

There are additional plugins and additional backends for the resource plugin.
For example, Trustee can integrate with [Azure Key Vault](kbs-backed-by-akv), [HashiCorp Vault KV](kbs-backed-by-vault-kv), or PKCS11 HSMs.
