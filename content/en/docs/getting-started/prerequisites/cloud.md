---
title: Cloud Hardware 
description: Confidential Containers on the Cloud 
weight: 15
categories:
- prerequisites
tags:
- peer pods
- cloud api adaptor
---
{{% alert title="Note" color="primary" %}}
If you are using bare metal confidential hardware, you can skip this section.
{{% /alert %}}

Confidential Containers can be deployed via confidential computing cloud offerings.
The main method of doing this is to use the [cloud-api-adaptor](https://github.com/confidential-containers/cloud-api-adaptor)
also known as "peer pods."

Some clouds also support starting confidential VMs inside of non-confidential VMs.
With Confidential Containers these offerings can be used as if they were bare-metal.
