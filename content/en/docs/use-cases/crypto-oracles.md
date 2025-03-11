---
date: 2025-03-11
title: "Switchboard Oracles: Securing Web3 Data with Confidential Containers"
description: "How Switchboard leverages Confidential Containers to create a secure, decentralized oracle network for Web3 applications"
author: Emanuele "Lele" Calo ([@eldios](https://github.com/eldios))
---

## Overview

Switchboard is building a decentralized oracle network that provides secure, reliable, and verifiable data for Web3 applications. By leveraging Confidential Containers (CoCo) with AMD SEV-SNP technology, Switchboard has created a robust infrastructure that protects sensitive oracle data from privileged attackers while ensuring verifiable attestation for blockchain applications.

## Challenge

Oracles serve as critical infrastructure in the blockchain ecosystem, providing external data to smart contracts. However, they face unique security challenges:

1. **Data Integrity**: Ensuring data remains unaltered from source to blockchain
2. **Confidentiality**: Protecting sensitive data from manipulation during processing
3. **Privileged Access Threats**: Defending against potential attacks from system administrators or cloud providers
4. **Verifiable Trust**: Providing cryptographic proof that data processing occurs in a secure environment

Traditional container solutions couldn't provide the level of isolation and attestation required for Switchboard's security model.

## Solution

Switchboard implemented Confidential Containers with AMD SEV-SNP to create a trusted execution environment for their oracle infrastructure:

- **Hardware-Level Memory Encryption**: Using AMD EPYC CPUs with SEV-SNP to encrypt memory contents, protecting data-in-use from privileged attackers
- **Remote Attestation**: Providing cryptographic verification that oracle code runs in a genuine, unmodified confidential environment
- **Confidential Containers Integration**: Building on Kata Containers' isolation while adding confidential computing capabilities
- **Decentralized Infrastructure**: Enabling trusted node partners to run Switchboard infrastructure while maintaining security guarantees

## Business Benefits

1. **Enhanced Security Posture**: Protection against infrastructure-level attacks, reducing the attack surface for oracle data
2. **Verifiable Trust**: Ability to cryptographically prove to users and partners that data processing occurs in a secure environment
3. **Competitive Advantage**: Offering higher security guarantees than traditional oracle solutions
4. **Simplified Compliance**: Helping meet regulatory requirements for sensitive data handling in Web3 applications
5. **Partner Network Growth**: Enabling secure decentralization through the Node Partners program

## Future Roadmap

Switchboard is expanding its confidential computing capabilities with planned support for Intel TDX, further broadening hardware compatibility options for node partners. This multi-architecture approach ensures the oracle network can scale securely across diverse infrastructure environments.

## Conclusion

By implementing Confidential Containers with AMD SEV-SNP, Switchboard has established a new security standard for Web3 oracles. This infrastructure provides the foundation for Switchboard's Node Partners program, launching on mainnet on March 11, 2025, enabling a truly decentralized and secure oracle network that Web3 applications can trust with their most sensitive data needs.
