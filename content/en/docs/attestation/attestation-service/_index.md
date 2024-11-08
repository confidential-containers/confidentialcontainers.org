---
title: Attestation Service (AS)
description: This service verifies TEE evidence
weight: 2
categories:
- docs
tags:
- docs
- attestation-service
---

The Attestation Service (AS or CoCo-AS) verifies hardware evidence. The AS was designed to be used with the [Key Broker Service (KBS)](../key-broker-service/) for Confidential Containers, but it can be used in a wide variety of situations. The AS can be used anytime TEE evidence needs to be validated.

Today, the AS can validate evidence from the following TEEs:

- Intel TDX
- Intel SGX
- AMD SEV-SNP
- ARM CCA
- Hygon CSV
- Intel TDX with vTPM on Azure
- AMD SEV-SNP with vTPM on Azure

## Overview

```
                                      ┌───────────────────────┐
┌───────────────────────┐ Evidence    │  Attestation Service  │
│                       ├────────────►│                       │
│ Verification Demander │             │ ┌────────┐ ┌──────────┴───────┐
│    (Such as KBS)      │             │ │ Policy │ │ Reference Value  │◄───Reference Value
│                       │◄────────────┤ │ Engine │ │ Provider Service │
└───────────────────────┘ Attestation │ └────────┘ └──────────┬───────┘
                        Results Token │                       │
                                      │ ┌───────────────────┐ │
                                      │ │  Verifier Drivers │ │
                                      │ └───────────────────┘ │
                                      │                       │
                                      └───────────────────────┘
```

The Attestation Service (AS) has a simple API. It receives attestation evidence and returns an attestation token containing the results of a two-step verification process. The AS can be consumed directly as a Rust crate (library) or built as a standalone service, exposing a REST or gRPC API. In Confidential Containers, the client of the AS is the Key Broker Service (KBS), but the evidence originates from the Attestation Agent inside the guest.

The AS has a two-step verification process.

1. Verify the format and provenance of evidence itself (e.g. check the signature of the evidence).
2. Appraise the claims presented in the evidence (e.g. check that measurements match reference values).

The first step is accomplished by one of the platform-specific [Verifier Drivers](#verifier-drivers).
The second step is driven by the [Policy Engine](#policy-engine) with help from the [Reference Value Provider Service (RVPS)](#reference-value-provider-service).
