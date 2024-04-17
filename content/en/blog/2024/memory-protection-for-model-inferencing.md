---
date: 2024-04-15
title: Memory Protection for AI ML Model Inferencing
linkTitle: Memory Protection for AI ML Model Inferencing
description: >
  How Confidential Containers (CoCo) can provide additional security to the model inferencing platform on Kuberentes.
author: "[Suraj Deshmukh](https://linktr.ee/surajssd) & [Pradipta Banerjee](https://www.linkedin.com/in/bpradipt/)"
tags:
- AI
- ML
categories:
- Use Case
---

## Introduction

With the rapid stride of artificial intelligence & machine learning and businesses integrating these into their products and operations, safeguarding sensitive data and models is a top priority. That's where Confidential Containers (CoCo) comes into picture. Confidential Containers:

- Provides an extra layer of protection for data in use.
- Helps prevent data leaks.
- Prevents tampering and unauthorized access to sensitive data and models.

By integrating CoCo with model-serving frameworks like KServe[^1], businesses can create a secure environment for deploying and managing machine learning models. This integration is critical in strengthening data protection strategies and ensuring that sensitive information stays safe.

[^1]: KServe Website: <https://kserve.github.io/website/>

## Model Inferencing

Model inferencing typically occurs on large-scale cloud infrastructure. The following diagram illustrates how users interact with these deployments.

![Model Inferencing](/img/blogs/memory-protection-for-model-inferencing/model-inferencing.png)

## Importance of Model Protection

Protecting both the model and the data is crucial. The loss of the model leads to a loss of intellectual property (IP), which negatively impacts the organization's competitive edge and revenue. Additionally, any loss of user data used in conjunction with the model can erode users' trust, which is a vital asset that, once lost, can be difficult to regain.

Additionally, reputational damage can have long-lasting effects, tarnishing a company's image in the eyes of both current and potential customers. Ultimately, the loss of a model can diminish a company's competitive advantage, setting it back in a race where innovation and trustworthiness are key.

## Attack Vectors against Model Serving Platforms

Model serving platforms are critical for deploying machine learning solutions at scale. However, they are vulnerable to several common attack vectors. These attack vectors include the following:

- **Data or model poisoning:** Introducing malicious data to corrupt the model's learning process.
- **Data privacy breaches:** Unauthorized access to sensitive data.
- **Model theft:** Proprietary or fine-tuned models are illicitly copied or stolen.
- **Denial-of-service attacks:** Overwhelming the system to degrade performance or render it inoperable.

The OWASP Top 10 for LLMs paper[^2] provides a detailed explanation of the different attack vectors.

[^2]: OWASP Top 10 for LLMs paper: <https://owasp.org/www-project-top-10-for-large-language-model-applications/assets/PDF/OWASP-Top-10-for-LLMs-2023-v1_1.pdf>

Among these attack vectors, our focus here is "model theft" as it directly jeopardizes the intellectual property and competitive advantage of organizations.

## Traditional Model Protection Mechanisms

Kubernetes offers various mechanisms to harden the cluster in order to limit the access to data and code. Role-Based Access Control (RBAC) is a foundational pillar regulating who can interact with the Kubernetes API and how. Thus ensuring that only authorized personnel have access to sensitive operations. API security mechanisms complements RBAC and acts as gatekeeper, safeguarding the integrity of interactions between services within the cluster. Monitoring, logging, and auditing further augment these defences by providing real-time visibility into the system's operations, enabling prompt detection and remediation of any suspicious activities.

![Traditional Model Protection Mechanisms on Kubernetes](/img/blogs/memory-protection-for-model-inferencing/traditional-model-protection.png)

Additionally, encrypting models at rest ensures that data remains secure even when not in active use, while using Transport Layer Security (TLS) for data in transit between components in the cluster protects sensitive information from interception, maintaining the confidentiality and integrity of data as it moves within the Kubernetes environment.
These layered security measures create a robust framework for protecting models against threats, safeguarding the valuable intellectual property and data they encapsulate.

_**But, is this enough?**_

## Demo: Read Unencrypted Memory

This video showcases how one can read the pod memory when it is run using the default runc[^8] or kata-containers[^9]. But using kata's confidential compute[^10] support we can avoid exposing the memory to the underlying worker node.

[^8]: runc: <https://github.com/opencontainers/runc>
[^9]: kata-containers: <https://katacontainers.io/>
[^10]: kata-cc <https://confidentialcontainers.org/docs/kata-containers/>

{{< youtube zZsB8S24CU4 >}}

## Confidential Containers (CoCo)

The Confidential Containers (CoCo) project aims at integrating confidential computing[^3] into Kubernetes, offering a transformative approach to enhancing data security within containerized applications. By leveraging Trusted Execution Environments (TEEs)[^4] to create secure enclaves for container execution, CoCo ensures that sensitive data and models are processed in a fully isolated and encrypted memory environment. CoCo not only shields the memory of applications hosting the models from unauthorized access but also from privileged administrators who might have access to the underlying infrastructure.

[^3]: Confidential Computing: <https://en.wikipedia.org/wiki/Confidential_computing>
[^4]: Trusted Execution Environments (TEEs): <https://en.wikipedia.org/wiki/Trusted_execution_environment>

As a result, it adds a critical layer of security, protecting against both external breaches and internal threats. The confidentiality of memory at runtime means that even if the perimeter defenses are compromised, the data and models within these protected containers remain impenetrable, ensuring the integrity and confidentiality of sensitive information crucial for maintaining competitive advantage and user trust.

## KServe

KServe[^1] is a model inference platform on Kubernetes. By embracing a broad spectrum of model-serving frameworks such as TensorFlow, PyTorch, ONNX, SKLearn, and XGBoost, KServe facilitates a flexible environment for deploying machine learning models. It leverages Custom Resource Definitions (CRDs), controllers, and operators to offer a declarative and uniform interface for model serving, simplifying the operational complexities traditionally associated with such tasks.

Beyond its core functionalities, KServe inherits all the advantageous features of Kubernetes, including high availability (HA), efficient resource utilization through bin-packing, and auto scaling capabilities. These features collectively ensure that KServe can dynamically adapt to changing workloads and demands, guaranteeing both resilience and efficiency in serving machine learning models at scale.

## KServe on Confidential Containers (CoCo)

In the diagram below we can see that we are running the containers hosting models in a confidential computing environment using CoCo. Integrating KServe with CoCo offers a transformative approach to bolstering security in model-serving operations. By running model-serving containers within the secure environment provided by CoCo, these containers gain memory protection. This security measure ensures that both the models and the sensitive data they process, including query inputs and inference outputs, are safeguarded against unauthorized access.

![KServe on CoCo](/img/blogs/memory-protection-for-model-inferencing/kserve-on-coco.png)
_Image Source[^5]_

[^5]: KServe Control Plane <https://kserve.github.io/website/latest/modelserving/control_plane/>

Such protection extends beyond external threats, offering a shield against potential vulnerabilities posed by infrastructure providers themselves. This layer of security ensures that the entire inference process, from input to output, remains confidential and secure within the protected memory space, thereby enhancing the overall integrity and reliability of model-serving workflows.

## Takeaways

Throughout this exploration, we've uncovered the pivotal role of Confidential Containers (CoCo) in fortifying data protection, particularly for data in use. CoCo emerges as a comprehensive solution capable of mitigating unauthorized in-memory data access risks. Model-serving frameworks, such as KServe, stand to gain significantly from the enhanced security layer provided by CoCo, ensuring the protection of sensitive data and models throughout their operational life cycle.

However, it's essential to recognize that not all components must operate within CoCo's protected environment. A strategic approach involves identifying critical areas where models and data are most vulnerable to unauthorized access and focusing CoCo's protective measures on these segments. This selective application ensures efficient resource utilization while maximizing data security and integrity.

## Further

In the next blog we will see how to deploy KServe on Confidential Containers for memory protection.

> **Note:**
> This blog is a transcription of the talk we gave at Kubecon EU 2024. You can find the slides on Sched[^6] and the talk recording on YouTube[^7].

[^6]: Fortifying AI Security in Kubernetes with Confidential Containers (CoCo) - Suraj Deshmukh, Microsoft & Pradipta Banerjee, Red Hat: <https://sched.co/1YeOx>
[^7]: Fortifying AI Security in Kubernetes with Confidential Containers (CoCo): <https://youtu.be/Ko0o5_hpmxI?si=JJRN9VMzvVzUz5vq>
