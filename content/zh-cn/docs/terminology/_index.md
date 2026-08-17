---
title: 术语对照表
description: Confidential Containers 文档术语中英文对照表
weight: 30
---

本文根据当前 CoCo 英文文档整理出常见的专业名称、关键术语与专有名词，便于中文文档翻译时保持用词一致。

说明：

- 组件名、项目名、协议名、工具名以及 `RuntimeClass` 名称通常保留英文。
- 中文列给出建议译法；对于不宜直译的项目名，采用“保留英文 + 中文解释”的方式。
- 该表会随着英文文档演进持续补充和调整。

## 核心项目与组件

| English | 中文建议 | 说明 |
| --- | --- | --- |
| Confidential Containers | 机密容器 | 项目正式名称 |
| CoCo | CoCo / 机密容器 | `Confidential Containers` 的简称 |
| Kata Containers | Kata Containers | CoCo 依赖的容器运行时项目 |
| Kata Agent | Kata Agent / Kata 代理 | 运行在客户机内的代理组件 |
| Kata Shim | Kata Shim | 宿主机侧连接容器运行时与客户机的组件 |
| PodVM | PodVM / Pod 虚拟机 | 承载机密工作负载的虚拟机 |
| Peer Pods | Peer Pods | 通过云 API 启动 PodVM 的部署模式 |
| guest-components | 客户机内组件 | 客户机内组件的集合 |
| image-rs | 镜像处理组件 | 客户机内镜像拉取与解包组件 |
| ocicrypt-rs | OCI 镜像解密组件 | 客户机内镜像解密组件 |
| confidential-data-hub | 机密数据中心 | 客户机内数据与机密访问组件 |
| CDH | 机密数据中心枢纽 | `confidential-data-hub` 的简称 |
| attestation-agent | 远程证明代理 | 客户机内远程证明客户端 |
| AA | 证明代理 | `attestation-agent` 的简称 |
| api-server-rest | API 服务代理 | Pod / 客户机内 API 访问代理 |
| cloud-api-adaptor | 云 API 适配器 | 通过云平台接口创建 PodVM 的组件 |
| CAA | 云 API 适配器 | `cloud-api-adaptor` 的简称 |
| Trustee | Trustee | CoCo 的证明与机密管理服务栈 |
| Key Broker Service | 密钥代理服务 | Trustee 中用于资源发放的服务 |
| KBS | 密钥代理服务 | `Key Broker Service` 的简称 |
| Attestation Service | 远程证明服务 | Trustee 中负责验证证据的服务 |
| AS | 证明服务 | `Attestation Service` 的简称 |
| Reference Value Provider Service | 参考值提供服务 | 提供参考测量值的服务 |
| RVPS | 参考值提供服务 | `Reference Value Provider Service` 的简称 |
| CoCo operator | CoCo Operator | 部署和管理 CoCo 的 Kubernetes Operator |
| nydus-snapshotter | Nydus 快照器 | 配合镜像拉取与快照管理的组件 |

## 安全、TEE 与证明术语

| English | 中文建议 | 说明 |
| --- | --- | --- |
| confidential computing | 机密计算 | 总体技术范畴 |
| Trusted Execution Environment | 可信执行环境 | 机密计算的核心执行环境 |
| TEE | 可信执行环境 | `Trusted Execution Environment` 的简称 |
| enclave | 安全飞地 / 隔离执行区 | 受保护的隔离执行环境 |
| Trusted Computing Base | 可信计算基 | 信任边界内的软硬件集合 |
| TCB | 可信计算基 | `Trusted Computing Base` 的简称 |
| Remote Attestation | 远程证明 | 验证平台或客户机可信性的过程 |
| attestation token | 远程证明令牌 | 远程证明结果的令牌化表示 |
| hardware evidence | 硬件证据 | 由 TEE 平台提供的证据 |
| reference values | 参考值 | 与实际度量结果比对的基准值 |
| verifier | 验证器 / 验证者 | 用于校验证据的组件 / 角色 |
| root of trust | 信任根 | 构建信任链的基础 |
| attestation policy | 远程证明策略 | 判定环境是否可信的策略 |
| resource policy | 资源策略 | 控制 KBS 是否发放资源的策略 |
| TCB claims | TCB 声明 | 从证据中提取的可信属性 |
| trust vector | 信任向量 | 结构化的证明结果表达 |
| EAR | 远程证明结果令牌 | Trustee 文档中使用的证明结果格式术语 |
| AR4SI | 远程证明结果互操作模型 | Trustee 使用的信任结果表达模型 |
| Init-Data | 初始化数据 | 注入客户机内的配置、策略和输入数据 |
| sealed secrets | 密封机密 | 保护机密数据的功能 |
| encrypted images | 加密镜像 | 经过加密保护的容器镜像 |
| signed images | 签名镜像 | 带有签名与完整性校验的镜像 |
| memory encryption | 内存加密 | 保护运行中数据的硬件能力 |
| GetResource | 获取资源请求 | 通过证明后向 KBS 获取机密资源的动作 |

## Kubernetes 与容器术语

| English | 中文建议 | 说明 |
| --- | --- | --- |
| CRI | 容器运行时接口 | `Container Runtime Interface` |
| CRD | 自定义资源定义 | `Custom Resource Definition` |
| OLM | Operator 生命周期管理器 | `Operator Lifecycle Manager` |
| Helm | Helm | Kubernetes 包管理工具 |
| Helm Chart | Helm Chart | Helm 安装包 |
| kubectl | kubectl | Kubernetes 命令行工具 |
| KServe | KServe | Kubernetes 上的模型推理平台 |
| ConfigMap | 配置映射 | Kubernetes 配置对象 |

## 平台、硬件与云环境

| English | 中文建议 | 说明 |
| --- | --- | --- |
| Intel TDX | Intel TDX | Intel Trusted Domain Extension TEE 技术 |
| Intel SGX | Intel SGX | Intel 安全飞地技术 |
| AMD SEV | AMD SEV | AMD 安全加密虚拟化技术 |
| AMD SEV-ES | AMD SEV-ES | 带寄存器状态保护的 SEV 扩展 |
| AMD SEV-SNP | AMD SEV-SNP | 带更强完整性保护的 AMD TEE 技术 |
| IBM Secure Execution | IBM-SE | IBM Z 平台的机密计算能力 |
| IBM Z | IBM Z | IBM 大型机平台 |
| LinuxONE | LinuxONE | IBM 企业级 Linux 平台 |
| s390x | s390x | IBM Z 使用的体系结构 |
| ARM CCA | ARM CCA | ARM 机密计算架构 |
| Hygon CSV | 海光 CSV | 海光平台机密计算技术 |
| Hygon DCU | 海光 DCU | 海光硬件加速器平台 |
| NVIDIA GPU | NVIDIA GPU | 英伟达GPU 机密计算相关平台 |
| Hopper | Hopper | NVIDIA GPU 架构代号 |
| Blackwell | Blackwell | NVIDIA GPU 架构代号 |
| Protected PCIe | 受保护的 PCIe | 英伟达 Hopper 一代多GPU互联总线保护技术 |
| vTPM | 虚拟 TPM | 虚拟化可信平台模块 |
| AWS | 亚马逊云 | 云平台 |
| Azure | 微软云 | 云平台 |
| GCP | Google Cloud | 云平台 |
| Alibaba Cloud | 阿里云 | 云平台 |
| OpenShift | OpenShift | 企业级 Kubernetes 发行版 |
| LibVirt | LibVirt | 本地虚拟化测试场景中使用的平台 |

## 标准、协议与工具

| English | 中文建议 | 说明 |
| --- | --- | --- |
| OCI | OCI 开放容器规范 | 容器镜像与运行时相关规范 |
| OCI Registry | OCI 镜像仓库 | 存储 OCI 镜像与工件的仓库 |
| SBOM | 软件物料清单 | `Software Bill of Materials` |
| SLSA | 供应链安全等级框架 | `Supply-chain Levels for Software Artifacts` |
| in-toto | in-toto 供应链证明框架 | 供应链证明格式与框架 |
| OPA | 开放策略代理 | `Open Policy Agent` |
| Rego | Rego 策略语言 | OPA 使用的策略语言 |
| ORAS | OCI 工件工具 | 用于操作 OCI Artifact 的 CLI |
| JWK | JSON Web Key | JSON Web 密钥格式 |
| JWT | JSON Web Token | 常见令牌格式 |
| PluginResponse | 插件响应 | KBS 外部插件文档中的接口名 |
| KBC | 密钥代理客户端 | `Key Broker Client` 的简称 |

## 扩展项目名与场景术语

| English | 中文建议 | 说明 |
| --- | --- | --- |
| BYOM | 自带机器模型 | `Bring Your Own Machine` |
| External Secrets Operator | 外部机密 Operator | 常简称 `ESO` |
| ESO | 外部机密 Operator | `External Secrets Operator` 的简称 |
| NIM | NVIDIA 推理微服务 | `NVIDIA Inference Microservices` |
| Switchboard | Switchboard | 相关博客中出现的项目名 |
| IRSA | 服务账户 IAM 角色机制 | AWS 场景常见术语 |
| EKS | Amazon Elastic Kubernetes Service | AWS Kubernetes 托管服务 |
| AKS | Azure Kubernetes Service | Azure Kubernetes 托管服务 |
| GKE | Google Kubernetes Engine | GCP Kubernetes 托管服务 |
| NGC API | NGC API | NVIDIA NGC 访问接口 |

## RuntimeClass 与字面标识符

以下名称通常不翻译，中文仅作解释。

| English | 中文建议 | 说明 |
| --- | --- | --- |
| `kata-qemu-coco-dev` | CoCo 开发运行时 | 非 TEE 开发与测试运行时 |
| `kata-qemu-coco-dev-runtime-rs` | CoCo Rust 开发运行时 | 基于 Rust 的开发运行时 |
| `kata-qemu-tdx` | TDX 运行时 | Intel TDX `RuntimeClass` |
| `kata-qemu-snp` | SNP 运行时 | AMD SEV-SNP `RuntimeClass` |
| `kata-qemu-sev` | SEV 运行时 | AMD SEV `RuntimeClass` |
| `kata-qemu-se` | Secure Execution 运行时 | IBM Secure Execution `RuntimeClass` |
| `kata-qemu-nvidia-gpu-tdx` | TDX + NVIDIA GPU 运行时 | GPU 机密计算场景 |
| `kata-qemu-nvidia-gpu-snp` | SNP + NVIDIA GPU 运行时 | GPU 机密计算场景 |
| `kata-remote` | 远程虚拟机运行时 | `Peer Pods` 或 `CAA` 场景 |
| `kata-clh` | Cloud Hypervisor 运行时 | 发行说明中出现的运行时名称 |
| `kata-clh-tdx` | Cloud Hypervisor TDX 运行时 | 发行说明中出现的运行时名称 |
| `kata-cc` | CoCo 运行时 | 部分策略示例中出现 |
| `io.containerd.cri.runtime-handler` | containerd 运行时处理器注解 | 用于选择运行时的注解 |
