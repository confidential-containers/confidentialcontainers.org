---
title: Hardware Requirements 
description: Hardware requirements for deploying Confidential Containers 
weight: 10
---

Confidential Computing is a hardware technology.
Confidential Containers supports multiple hardware platforms
and can leverage cloud hardware.
If you do not have bare metal hardware and will deploy Confidential Containers
with a cloud integration, continue to the cloud section.

You can also run Confidential Containers without hardware support
for testing or development.

The Confidential Containers operator, which is described in the following section,
does not setup the host kernel, firmware, or system configuration. 
Before installing Confidential Containers on a bare metal system, 
make sure that your node can start confidential VMs.

This section will describe the configuration that is required on the host.

Regardless of your platform, it is recommended to have at least 8GB of RAM and 4 cores
on your worker node.
