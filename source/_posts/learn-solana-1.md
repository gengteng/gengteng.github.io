---
title: 'Solana Developer’s Journey: Part 2 - Consensus Mechanism'
date: 2024-08-13 11:28:50
tags:
---

**Content Overview**
This section discusses the differences between the two major blockchains, Solana and Ethereum. Understanding their technical details and ecosystem differences is crucial for developers and ecosystem participants. This section provides an overall introduction to Solana, with specific details expanded in the following chapters. The discussion covers aspects such as consensus mechanism, transaction processing capabilities, transaction fees, smart contracts, accounts, programming languages, and developer friendliness.

**Section One: Consensus Mechanism**
- **Solana**: Utilizes a combination of Proof of History (PoH) and Proof of Stake (PoS). PoH records and verifies timestamps and block order, enabling nodes to reach consensus quickly. PoS is used for validator selection, ensuring network security and resistance to attacks.
- **Ethereum**: Completed the Ethereum 2.0 upgrade in September 2022, transitioning from Proof of Work (PoW) to Proof of Stake (PoS), significantly reducing energy consumption and enhancing security.

**Section Two: Transaction Processing Capability**
- **Solana**: Supports parallel processing by dividing transactions into subsets assigned to different validation nodes, significantly enhancing throughput. It processes over 2,000 transactions per second on average, maintaining low fees.
- **Ethereum**: Processes transactions sequentially, with a throughput of 15 to 30 transactions per second on average. Layer 2 and Rollup solutions enhance processing capability and reduce fees. The upcoming Proto-Danksharding upgrade is expected to further improve performance.

**Section Three: Transaction Fees (Gas Fees)**
- **Solana**: Calculates fees dynamically based on transaction complexity and size, with average fees below $0.01, around $0.00025, making it cost-effective for small transactions.
- **Ethereum**: Fees fluctuate based on network congestion. The current range for simple transfers is $1 to $10. The EIP-1559 upgrade introduced a new fee market mechanism to make fees more predictable.

**Section Four: Smart Contracts**
- **Solana**: Treats everything as an account, including smart contracts (referred to as programs). Programs are divided into executable accounts and data accounts, simplifying upgrades.
- **Ethereum**: Smart contracts include both logic code and state data, and once deployed, they do not support direct upgrades, only indirect upgrades through redeployment.

**Section Five: Accounts**
- **Solana**: Everything is an account, including program code, state data, and metadata. This single-account model supports parallel processing of multiple transactions, enhancing performance.
- **Ethereum**: Divides into Externally Owned Accounts (EOA) and Smart Contract accounts, offering a flexible and powerful decentralized application development platform.

**Section Six: Programming Language and Developer Friendliness**
- **Solana**: Supports multiple programming languages, including Rust and C, known for their high performance.
- **Ethereum**: Uses Solidity, a language designed for smart contract development, with a wealth of development tools and a large developer community, providing abundant resources for newcomers.

**Section Seven: Ecosystem and User Base**
- **Solana**: Rapidly building its ecosystem, presenting many opportunities despite being relatively new.
- **Ethereum**: Boasts the largest decentralized application (DApp) ecosystem and a vast user and developer community, offering strong network effects and a higher degree of decentralization.

Through this comparison, it is clear that each platform has its strengths and challenges. Ethereum's widespread adoption and mature ecosystem provide stability and trust, while Solana's high performance and low costs offer advantages for applications requiring these features.

> **Course address**: https://www.hackquest.io/