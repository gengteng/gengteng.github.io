---
title: 'Solana Developer’s Journey: Part 1 - The Evolution and Core Concepts'
date: 2024-08-13 11:22:42
tags:
---

**Content Overview**
Before we start learning to build on Solana, let’s first understand its history.

**History of Solana**

**Key Events and Figures**

- **November 2017**: Anatoly Yakovenko published a whitepaper introducing the concept of "Proof of History" (PoH), designed to solve time synchronization issues among mutually untrusting computers. With experience in designing distributed systems at Qualcomm, Mesosphere, and Dropbox, Anatoly recognized that reliable clocks could simplify network synchronization, enabling fast network operations constrained only by bandwidth.

- **Background and Motivation**: Anatoly observed that blockchains like Bitcoin and Ethereum, lacking a clock, could achieve only 15 transactions per second. He envisioned the need for peak transaction speeds of 65,000 transactions per second for a globally decentralized payment system like Visa. Realizing that solving time inconsistency among computers was crucial, he applied years of research in distributed systems to the blockchain domain, which not only improved efficiency but also increased speed by orders of magnitude.

- **Early Development**: Initially, Anatoly implemented the Solana prototype in C language on his GitHub. Greg Fitzgerald, who had previously collaborated with Anatoly at Qualcomm, encouraged him to rewrite the project in Rust. Within two weeks, Anatoly successfully migrated the project to Rust and named it Loom.

- **February 13, 2018**: Greg Fitzgerald began creating an open-source prototype based on Anatoly's whitepaper.

- **February 28, 2018**: Greg released the first version capable of verifying and processing over 10,000 signed transactions in half a second. Shortly after, another former Qualcomm colleague, Stephen Akridge, demonstrated how to enhance processing speed by using graphics processors for signature verification. Anatoly then invited Greg, Stephen, and three others to co-found a company named Loom.

- **Renaming to Solana**: Around the same time, another Ethereum-based project called Loom Network emerged, causing occasional confusion between the two projects. The Loom team decided to rename their project. Eventually, they chose the name "Solana" in homage to their three years of work and surfing in Solana Beach, Northern San Diego.

- **March 28, 2018**: The team created Solana's GitHub and officially named Greg Fitzgerald's prototype Solana.

**Summary**
The creation of Solana was the result of years of effort by Anatoly Yakovenko and his team, drawing on their extensive experience in distributed systems and blockchain technology. Key figures include Anatoly Yakovenko, Greg Fitzgerald, and Stephen Akridge. Through the innovative "Proof of History" mechanism and efficient development practices, they established Solana as a blockchain platform known for its high throughput and low latency.

> **Course address**: https://www.hackquest.io/