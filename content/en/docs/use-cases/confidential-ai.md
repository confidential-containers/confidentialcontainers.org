---
title: Confidential AI
description: Confidential Containers for AI Use Cases
categories:
- use cases
tags:
- AI 
- ML
weight: 60
---
 
## Federated Learning

![Federated Learning](/img/FederatedLearning.png)

### What is federated Learning 

Federated Learning (FL) is a decentralized machine learning approach where multiple participants (such as organizations, edge devices, or distributed servers) collaboratively train a model without sharing their raw data. Instead of centralizing data, each participant trains a local model and shares only model updates with a central aggregator, preserving data privacy and reducing communication overhead.

FL is widely applied in domains like healthcare, finance, transportation, and edge computing, where concerns on data privacy, cost of moving data, regulatory compliance and security are critical.

### Confidential Federated AI with confidential computing

Although Federated Learning (FL) doesn't move the data, there are still data privacy issues with model update. There are several privacy enhancement technologies (PETs) are available: example Differential Privacy and Homomorphic Encryption. 

By leveraging Confidential Computing Trusted Execution Env. (TEE), we are adding additional security option or layer for the FL. FL + Confidential Computing can be called Confidential Federated AI. 

### Confidential Federated AI Use Cases and Requirements

Unlike most of AI Inference or RAG, or clean room use cases, Federated learning could have multiple Confidential TEEs.  The use cases and requirements are different than single TEE case.

 We have identified three common use cases

 ####  Building Explicit Trust

Since FL requires  multiple participants (banks, hospitals etc.) to collaborate, they must trust each other.  Traditionally, such trust is based on: business/legal contracts, IT security team verifications, such trust can be considered implicit trust.  As participants need to trust the IT infrastructure and data scientists on all sides to be honest, no ill intentions and make no mistakes. 

If the participants really concerns the security, CC will provide a much strong guarantee with "explicit trust" by leveraging CC Attestation. 

**"implicit trust" ==> "explicit trust"**
   
##### Requirements for Attestation 

1.  We need to be able to verify the other participants are trust worthy with specified CC policy. 

This is cross-verification via attestation, the policy is cross-verification polices. 
 
For example, in FL system, we have a FL server and several FL clients. FL Server needs to be able check FL Clients are trust worthy; FL clients want to make sure the FL Server is trust worthy as well. 

* **Approach 1**: CoCo Allows App get Attestation Token

FL Server and FL Client exchange the attestation tokens and then independently verify the received token via different attestation services. 

Currently, **CoCo doesn't provide a way for Application to get the attestation token**,  therefore this approach currently doesn't work in CoCo. 

* **Approach 2**:  CoCo Trust will provide API to perform the verification 
      
For example,  instead FL server and FL Client exchange attestation tokens,  FL Server will ask CoCo Trustee with instruction like:

"please verify Client with ID = 123 is trust worthy based on policy 1"
            
 ``` 
                fl_server.get_trustee().verify( object_id = 123, policy=policy_1.json) 
          
 ```

Similarly, FL Client will ask CoCo Trustee to do the same 
"please verify FL Server with ID = 456 is trust worthy based on policy 2"
            
``` 
                fl_client.get_trustee().verify( object_id = 456, policy=policy_2.json) 
          
```
In other words, in CoCo environment. Please provide one of ways to do it.        either you let me do it myself ( provide me with token) or you do it for me. 

2. Attestation must be allow to be verified ad hoc or periodical 
        
   
####  Secure Aggregation

The 2nd use case is secure aggregation. This similar to the clean-room use case, where the client's model is send to the aggregator ( usually is in FL server), where the model is aggregated.  In a simplest case, Client trust each other, but not trust server, as the model can potentially inverted to find private data. 

In this case, we can use Confidential Computing TEE  (CoCo) to lock down the access to the TEE. 

  
####  Model IP Protection

The 3rd Use case is avoid model is being copied away during fine-tuning at Client side. 

In this case, we can lock down the FL client side with CC TEE and other techniques. 

### Summary 

#### Attestations
- Federated Learning requests multi-SDK attestation
- FL Servers needs to verify all clients' trustworthiness
- Attestation at different points, self and cross verifications via attestation service

#### CC Policies:
- Bootup policy – provided by hardware vendor
- Self-verification CC policy – user defined (for example the policy specifies the pod needs a GPU, but the host has no GPU, then self-verification should fail)
- Cross-verification CC policy  -- user defined

#### Protected Assets
- **Code**: Training code (client), aggregation code (server)
- **Data**: input data
- **Model**:  initial model, intermediate model, output model
- **Workspace**: check points, logs, temp data, output directory

#### Key Broker Service
- Key management depends on user case, global model ownership, key release management process

#### Bootstrap:
- Need a process to generate the keys, policies and input them into key-vault to avoid tempering

#### Concerns when using federated learning
- Trustworthy of the participants
- Execution code tampering
- Model tampering
- Model inversion attack
- Model theft during training
- Private data leak 

#### CoCo will not protect if
- code is already flawed at rest
- data is already poisoned at rest
- model is already poisoned at rest

## Multi-Party Computing

![Multi-Party Computing](/img/MultiPartyCompute.png)

Source: [https://uploads-ssl.webflow.com/63c54a346e01f30e726f97cf/6418fb932b58e284018fdac0_OC3%20-%20Keith%20Moyer%20-%20MPC%20with%20CC.pdf](https://uploads-ssl.webflow.com/63c54a346e01f30e726f97cf/6418fb932b58e284018fdac0_OC3%20-%20Keith%20Moyer%20-%20MPC%20with%20CC.pdf)


### Requirements
- Attestation-cross-verification
  - In MPC or FL cases, we need to explicitly verify all participants to be trustworthy according to the cross verification CC policy. 
- Periodic self-check test and cross-verification
  - a long running workload can ensure that the TEE is still valid
- Ad hoc attestation verification for given participation node
  - From CLI or API, application wants to know what’s the current status (trustworthiness) of the specified node
- Attestation audit report 
  - each party would like to get details on when attestation was performed by the TEE and relevant details
- Connecting to multiple KBSes
  - the workload needs to connect to different KBS each belonging to a specific party for getting access to the keys
- Support multiple devices
  - One participant may has only one type of device (CPU), but need to verify other participant’s devices including different CPU and GPU

## Retrieval Augmented Generation Large Language Models (RAG LLM)

![RAG LLM](/img/RAG_LLMS.svg)

Further RAG Use Case Information 
- [NVIDIA Generative AI Examples](https://github.com/NVIDIA/GenerativeAIExamples)

### Threat Model
Potential Threats: [OWASP Top 10 for LLMs](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

![RAG Threat Model](/img/RAG_ThreatModel.svg)
