---
title: Confidential Containers
---

{{< blocks/cover image_anchor="top" height="auto" >}}
<p class="fw-bold fa-3x">
Welcome to Confidential Containers!
</p>
<br><br><br><br>
{{< blocks/link-down color="info" >}}
{{< /blocks/cover >}}

<!-- Note: The content here is mostly copied / adapted from: https://github.com/confidential-containers/.github/blob/main/profile/README.md. This page shows up when one goes to: https://github.com/confidential-containers/ -->

{{% blocks/section %}}

The goal of the CoCo project is to standardize confidential computing at the pod level and simplify its consumption in Kubernetes. This enables Kubernetes users to deploy confidential container workloads using familiar workflows and tools without extensive knowledge of the underlying confidential computing technologies.
{.h4 .text-left}
<br>

With CoCo, you can deploy your workloads on shared infrastructure yet significantly reduce the risk of unauthorized entities accessing your workload data and extracting your secrets.
{.h4 .text-left}
<br>

Confidential Containers is an open source community working to enable cloud native confidential computing by leveraging [Trusted Execution Environments](https://en.wikipedia.org/wiki/Trusted_execution_environment) to protect containers and data.
{.h4 .text-left}

{{% /blocks/section %}}

{{% blocks/lead height="auto" %}}
<a class="btn btn-lg btn-primary me-3 mb-4" href="/docs/">
  Learn More <i class="fas fa-arrow-alt-circle-right ms-2"></i>
</a>
<a class="btn btn-lg btn-success me-3 mb-4" href="https://github.com/confidential-containers">
  Download <i class="fab fa-github ms-2 "></i>
</a>

{{% /blocks/lead %}}

<!-- Note: The content here is mostly copied / adapted from: https://github.com/confidential-containers/.github/blob/main/profile/README.md. This page shows up when one goes to: https://github.com/confidential-containers/ -->

{{% blocks/section type="row" %}}

Goals
{.h1 .text-center}

{{% blocks/feature icon="fa-microchip" title="Multiple TEEs" %}}
Support for multiple Trusted Execution Environments (TEEs) and hardware platforms

Please follow this space for updates!
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-cubes" title="Containers" %}}
Transparent deployment of unmodified containers
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-cloud" title="Cloud Service Providers (CSP)" %}}
A trust model which separates CSPs from guest applications
{{% /blocks/feature %}}

{{% /blocks/section %}}

{{% blocks/section type="row" %}}

{{% blocks/feature icon="fa-shield" title="Application Security" %}}
Allow cloud native application owners to enforce application security requirements
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-key" title="Privilege" %}}
Least privilege principles for the Kubernetes Cluster administration capabilities which impact delivering Confidential Computing for guest application or data inside the TEE.
{{% /blocks/feature %}}

{{% /blocks/section %}}

{{% blocks/section type="row" %}}

Community
{.h1 .text-center}

{{% blocks/feature icon="fab fa-github" title="Contributions welcome!"
    url="<https://github.com/confidential-containers/confidentialcontainers.org>" %}}
We do a [Pull Request](https://github.com/confidential-containers/confidentialcontainers.org/pulls)
contributions workflow on **GitHub**. New users are always welcome!
{{% /blocks/feature %}}

{{% blocks/feature icon="fab fa-slack" title="We are on CNCF Slack!" url="<https://slack.cncf.io>" %}}
Join channel #confidential-containers by getting invitation for the [CNCF slack](https://slack.cncf.io).
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-video-camera" title="Weekly Meetings" url="<https://docs.google.com/document/d/1E3GLCzNgrcigUlgWAZYlgqNTdVwiMwCRTJ0QnJhLZGA/>" %}}
Check out our previous meetings and join our future ones.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-users" title="Community Guidelines" url="<https://github.com/confidential-containers/confidential-containers>" %}}
How to contribute, style guides, governance...
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-file" title="Code of Conduct" url="<https://github.com/cncf/foundation/blob/master/code-of-conduct.md>" %}}
We follow the CNCF Code of Conduct.
{{% /blocks/feature %}}

{{% /blocks/section %}}
