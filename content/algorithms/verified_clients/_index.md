---
title: Filecoin Plus
weight: 8
dashboardWeight: 2
dashboardState: wip
dashboardAudit: wip
dashboardTests: 0
---

# Filecoin Plus

Filecoin Plus is a layer of social trust designed to maximize the amount of useful storage on Filecoin. While a storage miner may choose to forgo deal payments and self-deal to fill their storage and earn block rewards, this is not as valuable to the economy and should not be heavily subsidized. However, in practice, it is impossible to tell useful data apart from encrypted zeros. Filecoin Plus pragmatically solves this problem through social trust and validation. The program operates through a decentralized network of Notaries who allocate DataCap to clients, enabling them to make deals that carry a 10x quality multiplier.

## Roles and Responsibilities

### Root Key Holders
Root Key Holders are signers to a multisig on chain with the power to grant and remove Notaries. They act as executors for decisions made by community governance, requiring a majority to sign for any action.

### Notaries

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0012
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0012.md
    description: Enabled DataCap top-ups to existing client addresses without requiring full depletion.
-->

Notaries form a decentralized, globally distributed network of entities that confirm the useful storage demand of Filecoin Plus clients. They are entrusted with DataCap to allocate to clients based on trust and verification. When a Notary evaluates and affirms a client's demand to have real data stored, that client receives a DataCap allocation. 

Since FIP-0012, Filecoin Plus clients can receive additional DataCap allocations to the same address at any time, without needing to fully deplete their existing balance. When topping up an existing client, the new allocation is added to their current DataCap balance. Notaries perform due diligence to ensure clients are not maliciously exploiting the system and should check existing allocations before approving additional DataCap.

### Filecoin Plus Clients
Clients are active participants with DataCap allocation for their use cases. They can use DataCap to incentivize miners to provide additional features and service levels. Clients must deploy DataCap responsibly in accordance with program principles.

## Technical Implementation

The Filecoin Plus mechanism interfaces with the on-chain protocol through the Verified Registry Actor. When clients make storage deals using their DataCap, these deals receive a 10x quality multiplier, providing greater quality-adjusted power to miners who store Filecoin Plus data.

Storage demand on the network shapes the storage offering provided by miners. With the 10x sector quality multiplier for Filecoin Plus deals, clients play a crucial role in shaping the quality of service, geographic distribution, degree of decentralization, and consensus security of the network. All participants - Root Key Holders, Notaries, and Filecoin Plus clients - must be cognizant of the value and responsibility that come with their roles.

## Governance

The Filecoin Plus program is governed through community-driven processes, with different layers of governance:
- **Principles**: Core values and goals (modified only through FIPs)
- **Mechanisms**: Specific implementations of principles
- **Operations**: Day-to-day processes and guidelines
- **Markets**: Dynamic ecosystem interactions

For detailed operational guidelines and to participate in governance, see the [Filecoin Plus Governance repository](https://github.com/filecoin-project/notary-governance).