---
title: "NameCoin on Peerster: A Blockchain-Based Decentralized DNS Implementation"
excerpt: "This project explores the design and implementation of a decentralized DNS system using blockchain and a gossip-based peer protocol. Built on top of Peerster, the system supports secure domain registration, updates, transfers, and resolution with robust anti-entropy synchronization and Proof-of-Work consensus. The project evaluates network resilience, consensus reliability, and mining efficiency."
collection: course_projects
share: false
related: false
reporturl: "../../files/NameCoin.pdf"

---

## Problem Statement & Motivation

The traditional Domain Name System (DNS) is centralized, exposing it to censorship, authority abuse, and single points of failure. Blockchain-based decentralized DNS (dDNS) solutions aim to distribute control, enhance security, and promote fault tolerance. This project implements a **blockchain-powered DNS service** over the **Peerster** protocol—initially designed for gossip and anti-entropy synchronization.

## Our Method

The system extends a prior gossip-layer implementation with a full **four-layer architecture**:

### 1. **Peerster Layer**
- Enables decentralized communication and anti-entropy peer synchronization.

### 2. **Blockchain Layer**
- Handles domain registration, updates, transfers, and expiration using **Proof-of-Work**.
- Implements fork resolution, block validation, and chain adoption logic.

### 3. **Transaction Layer**
- Manages UTXO-based domain transactions: "new name", "first update", "update", and "transfer".
- Ensures transaction integrity through cryptographic validation.

### 4. **DNS Layer**
- Interfaces with clients for domain resolution queries and returns IP ownership data from the chain.

## Implementation Highlights

- Messages are exchanged between miners and clients for transaction submission, block mining, and DNS queries.
- Forks are handled using rollback-and-cache mechanisms ensuring blockchain consistency.
- An **indexed blockchain pool** stores out-of-order chain fragments, resolving synchronization issues in late-joining peers.
- A simple identity system maps IPs to public keys due to absence of a Certificate Authority.

## Evaluation & Testing

We performed **unit**, **integration**, and **performance tests**, including:

- **Transaction Throughput and Latency** under variable client/miner setups.
- **Consensus Stability** in forked or churned networks.
- **Network Overhead Analysis**, including message size, type distribution, and bandwidth.

## Results

- Throughput increases with more miners but degrades with higher PoW difficulty.
- The system is resilient to churn and Byzantine conditions via anti-entropy and fork handling.
- DNS query handling and domain transactions maintain consistency across peers.

## Limitations & Future Work

- No formal CA; identity is manually assigned at init.
- Duplicate domain name detection during concurrent registration is limited due to salted hashes.
- Current implementation trusts buyers in transfers—requiring seller acknowledgment would enhance security.
- Consensus fork resolution could benefit from dynamic message sizing based on bandwidth.

## Project Report

- **Authors**: Oussama Gabouj, Rassene M’sadaa, Ahmed Abdelmalek  
- **Course**: CS438 - Decentralized Systems  
- **Institution**: EPFL  
