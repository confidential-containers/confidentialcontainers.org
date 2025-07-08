---
title: Reference Values
description: Managing Reference Values with the RVPS 
weight: 50
categories:
- attestation
tags:
- trustee
- attestation
- RVPS
---

Reference values are used by the attestation service, to generate an attestation token.
More specifically the attestation policy specifies how reference values are compared to TCB claims
(extracted from the HW evidence by the attesters).

## Provisioning Reference Values

There are multiple ways to provision reference values.

### KBS Client

The KBS Client can be used to set reference values.
When using the KBS Client to set reference values,
the sample reference value extractor is used.
This must be enabled in the RVPS (it is by default).
When using the KBS Client for reference values
the requests are proxied through the trusted admin interface.

To set a reference value, use a command like this.
```bash
./kbs-client config --auth-private-key <admin-private-key> set-sample-reference-value <rv-name> <rv-value>
```

By default, the reference value will be added as a list containing one string value.
If you would like to add the reference value as a single value without a list,
you can use the `--as-single-value` flag.
To add numeric reference values, you can use the `--as-integer` flag.
To add boolean reference values, use the `--as-bool` flag.
These flags can be combined with `--as-single-value`.

To register more complex reference values (any JSON types are supported by the RVPS),
use the RVPS Tool described below.

To view reference values, you can use the following command.
```bash
./kbs-client config --auth-private-key <admin-private-key> get-reference-values
```


### RVPS Tool

The RVPS tool supports more complex reference value operations
and is not coupled to the KBS admin credentials/persona.
The RVPS tool can be used to ingest reference values using any
extractor (not just the sample extractor).
The RVPS tool connects to the RVPS directly, which must be
accessible from wherever the tool is being run.

You can build the RVPS Tool alongside the RVPS.
```bash
git clone https://github.com/confidential-containers/trustee
cd trustee/rvps
make
```

The RVPS generally expects reference values to be delivered via signed messages.
For testing, you can create a sample message that is not signed.

```bash
cat << EOF > sample
{
    "test-binary-1": [
        "reference-value-1",
        "reference-value-2"
    ],
    "test-binary-2": [
        "reference-value-3",
        "reference-value-4"
    ]
}
EOF
provenance=$(cat sample | base64 --wrap=0)
cat << EOF > message
{
    "version" : "0.1.0",
    "type": "sample",
    "payload": "$provenance"
}
EOF
```

You can then provision this message to the RVPS using the RVPS Tool
```bash
rvps-tool register --path ./message --addr <address-of-rvps>
```

If you've deployed Trustee via docker compose, the RVPS address should be `127.0.0.1:50003`.

You can also query which reference values have been registered.
```bash
rvps-tool query --addr <address-of-rvps>
```

This provisioning flow is being refined.

### Operator

If you are using the Trustee operator, you can provision reference values through Kubernetes.
```bash
kubectl apply -f - << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: rvps-reference-values
  namespace: kbs-operator-system
data:
  reference-values.json: |
    [
    ]
EOF
``` 
