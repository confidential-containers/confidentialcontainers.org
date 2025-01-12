---
title: Reference Value Provider Service (RVPS)
description: This service manages reference values used to verify TEE evidence
weight: 2
categories:
- docs
tags:
- docs
- rvps
---

Reference Value Provider Service (RVPS) is a component to receive software supply chain provenances / metadata, verify them and extract the reference values. All the reference values are stored inside RVPS. When Attestation Service (AS) queries specific software claims, RVPS will response with related reference values.

## Architecture

RVPS contains the following components:

- **Pre-Processor:** Pre-Processor contains a set of *wares (like middleware). These wares can process the input **Message** and then deliver it to the **Extractors**.

- **Extractors:** Extractors has sub-modules to process different type of provenance. Each sub-module will consume the input **Message**, and then generate an output Reference Value.

- **Store:** Store is a trait object, which can provide key-value like API. All verified reference values will be stored in the **Store**. When requested by Attestation Service (AS), related reference value will be provided.

## Message Flow

The message flow of RVPS is like the following figure:

![](https://raw.githubusercontent.com/confidential-containers/trustee/main/rvps/diagrams/rvps.svg)

### Message

A protocol helps to distribute provenance of binaries. It will be received and processed by RVPS, then RVPS will generate Reference Value if working correctly.

```json
{
    "version": <VERSION-NUMBER-STRING>,
    "type": <TYPE-OF-THE-PROVENANCE-STRING>,
    "provenance": #provenance,
}
```

- `"version"`: This field is the version of this message, making extensibility possible.
- `"type"`: This field specifies the concrete type of the provenance the message carries.
- `"provenance"`: This field is the main content passed to RVPS. This field contains the payload to be decrypted by RVPS. The meaning of the provenance depends on the type and concrete **Extractor** which process this.

### Trust Digests

It is the reference values really requested and used by Attestation Service to compare with the gathered evidence generated from HW TEE. They are usually digests. To avoid ambiguity, they are named `trust digests` rather than `reference values`.
