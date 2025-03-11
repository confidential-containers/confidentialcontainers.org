---
date: 2025-03-11
title: "How Switchboard Oracles Leverage Confidential Containers for Next-Generation Web3 Security"
description: "Switchboard integrates Confidential Containers with AMD SEV-SNP to create a groundbreaking security model for their decentralized oracle network"
lead: "Enhancing oracle security through confidential computing and hardware-based memory encryption"
author: Emanuele "Lele" Calo ([@eldios](https://github.com/eldios))
---

The Web3 ecosystem depends on oracles to bridge on-chain smart contracts with off-chain data. However, this critical infrastructure has historically faced significant security challenges. Today, we're excited to share how Switchboard has integrated Confidential Containers (CoCo) with AMD SEV-SNP technology to create a groundbreaking security model for their decentralized oracle network.

## The Oracle Security Challenge

Oracles serve as the eyes and ears of blockchain applications, providing essential data that powers DeFi protocols, prediction markets, gaming applications, and more. With billions of dollars depending on accurate oracle data, security isn't optional—it's existential.

Traditional oracle solutions face several key vulnerabilities:

- **Infrastructure-level attacks** from privileged users like cloud administrators
- **Memory inspection** that could reveal sensitive data during processing
- **Lack of verifiable attestation** to prove computations occur in a secure environment
- **Centralization risks** when security requirements limit who can run infrastructure

## Enter Confidential Containers

Confidential Containers extends the isolation properties of Kata Containers with confidential computing capabilities. For Switchboard, this combination provides the ideal foundation for their security-critical oracle infrastructure.

### How It Works

Switchboard's implementation uses AMD EPYC processors with SEV-SNP (Secure Encrypted Virtualization-Secure Nested Paging) technology to create hardware-enforced trusted execution environments. This technology:

1. **Encrypts all memory contents** of the container, protecting data-in-use from privileged attackers
2. **Provides cryptographic attestation** that oracle code runs in a genuine, unmodified environment
3. **Maintains performance** while adding security guarantees
4. **Enables secure decentralization** through their Node Partners program

The integration builds on Switchboard's previous work with Kata Containers, adding the critical confidential computing layer that protects against an entire class of infrastructure-level attacks.

## Implementation Journey

While Confidential Containers offers tremendous security benefits, implementing it for production use required collaboration with the CoCo community. The Switchboard team worked through several challenges:

- Extending CoCo's capabilities through custom forks to meet specific oracle security requirements
- Optimizing performance for the low-latency needs of oracle data delivery
- Creating verification mechanisms to ensure node partners run the attested confidential environments
- Developing operational procedures for secure key management within the confidential environment

"The Confidential Containers community has been instrumental in helping us implement this technology for our production environment," says the Switchboard team. "Their guidance helped us navigate the complexities of confidential computing while maintaining the performance our oracle network requires."

## Technical Architecture

At a high level, Switchboard's implementation includes:

1. **AMD SEV-SNP enabled hosts** running on bare metal with EPYC CPUs
2. **Confidential Containers runtime** providing the trusted execution environment
3. **Remote attestation service** verifying the authenticity of oracle environments
4. **Secure data processing pipeline** that handles oracle requests within the protected memory
5. **Blockchain integration layer** that submits verified data on-chain

This architecture ensures that from the moment data enters the oracle system until it's cryptographically signed and submitted on-chain, it remains protected by hardware-enforced security boundaries.

## The Road Ahead

Switchboard's integration with Confidential Containers represents just the beginning of their confidential computing journey. Future plans include:

- Support for Intel TDX (Trust Domain Extensions) to broaden hardware compatibility
- Enhanced attestation mechanisms for multi-party verification
- Performance optimizations for specific oracle workloads
- Expanded node partner program to further decentralize the network

## Conclusion

The integration of Switchboard Oracles with Confidential Containers demonstrates how cutting-edge confidential computing technology can address the unique security challenges of Web3 infrastructure. By protecting sensitive oracle operations with hardware-enforced memory encryption and attestation, Switchboard is setting a new standard for oracle security.

As the Switchboard Node Partners program goes live on March 11, 2025, this technology will enable a truly decentralized oracle network that doesn't compromise on security—a critical advancement for the entire Web3 ecosystem.

---

*For more information about Confidential Containers, visit [confidentialcontainers.org](https://confidentialcontainers.org). To learn more about Switchboard's oracle network, check out their [documentation](https://github.com/switchboard-xyz/infra-external).*
