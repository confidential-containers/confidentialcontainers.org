---
title: CoCo without Hardware
description: Testing and development without hardware 
weight: 10
categories:
- prerequisites
tags:
- Non-tee
---

For testing or development, Confidential Containers can be deployed without
any hardware support. 

This is referred to as a `coco-dev` or `non-tee`.
A `coco-dev` deployment functions the same way as Confidential Containers
with an enclave, but a non-confidential VM is used instead of a confidential VM.
This does not provide any security guarantees, but it can be used for testing.

No additional host configuration is required as long as the host supports virtualization.
