---
title: Virtual Machine
description: VM - Virtual Machine
bookCollapseSection: true
weight: 3
dashboardWeight: 2
dashboardState: reliable
dashboardAudit: n/a
dashboardTests: 0
---

# Virtual Machine

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0030
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0030.md
    description: Introduced the Filecoin Virtual Machine (FVM) with WASM-based execution and user-programmable actors.
-->

An Actor in the Filecoin Blockchain is the equivalent of the smart contract in the Ethereum Virtual Machine.

The Filecoin Virtual Machine (VM) is the system component that is in charge of execution of all actors code. Since FIP-0030, the FVM is a WASM-based execution environment that supports both built-in actors and user-programmable actors, enabling general smart contract functionality on Filecoin. Execution of actors on the Filecoin VM (i.e., on-chain executions) incur a gas cost.

## Overview

The FVM provides:
- **WASM Execution**: A WebAssembly-based runtime for executing actor code
- **User Programmability**: Support for deploying and running arbitrary user-defined actors
- **Built-in Actors**: Continued support for Filecoin's system actors (storage market, miner, etc.)
- **IPLD Integration**: Native support for IPLD data structures and content-addressed storage
- **Syscall Interface**: A comprehensive set of system calls for actor-to-system interactions

Any operation applied (i.e., executed) on the Filecoin VM produces an output in the form of a _State Tree_ (discussed below). The latest _State Tree_ is the current source of truth in the Filecoin Blockchain. The _State Tree_ is identified by a CID, which is stored in the IPLD store.
