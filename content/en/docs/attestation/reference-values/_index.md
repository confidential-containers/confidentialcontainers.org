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

### RVPS Tool

The RVPS provides a client tool for providing reference values.
This is separate form the KBS Client because the reference value provider
might be a different party than the administrator of Trustee.

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
