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
  - fip: FIP-0031
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0031.md
    description: Switched to non-programmable FVM with content-addressed Code CIDs and introduced system actor state.
  - fip: FIP-0050
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0050.md
    description: Restricted built-in actor method invocation and defined stable public APIs for user actors.
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

## Actor Code CIDs

Every actor in the state tree specifies a Code CID that identifies its executable code and serves as its type designator. Since FIP-0031, these are content-addressed CIDs computed over the actor's WASM bytecode:

```
CodeCid = Cid(IPLD_RAW_CODEC, Mh(BLAKE2B-256, wasm_bytecode))
```

Prior to FIP-0031, the network used synthetic CIDs of the form `fil/$actor_version/$actor_type`. The transition to content-addressed CIDs improved security and enabled proper content verification.

### Built-in Actors

The canonical implementation of Filecoin's built-in actors is maintained at [`filecoin-project/builtin-actors`](https://github.com/filecoin-project/builtin-actors). The build process produces a CARv1 archive containing all actor WASM bytecode, which clients import on startup.

Since FIP-0031, the system actor (f00) maintains a registry of built-in actor Code CIDs in its state, enabling dynamic actor version management.

## Method Invocation and Access Control

Since FIP-0050, the FVM enforces strict access control on built-in actor methods to maintain stability for user-programmed actors:

### Method Number Conventions

Following FRC-0042, method numbers are organized as:
- **0 to 2^24-1**: Internal methods restricted to built-in actors
- **2^24 and above**: Public exported methods callable by any actor

### Access Restrictions

When invoking a method on a built-in actor:
1. If the method number is ≥ 2^24, it is considered "exported" and callable by any actor
2. If the method number is < 2^24, the FVM checks the caller's code CID:
   - Built-in actors can invoke these internal methods
   - User-programmed actors (including EVM actors) receive an error

This separation ensures that:
- Internal implementation details of built-in actors can evolve without breaking user actors
- User actors have access to a stable, well-defined API that won't change
- The protocol can be upgraded while maintaining backward compatibility

### Public APIs

Each built-in actor exports specific methods for public use:
- **Account Actor**: AuthenticateMessage, UniversalReceiverHook
- **Storage Market Actor**: AddBalance, WithdrawBalance, PublishStorageDeals, GetBalance, GetDealDataCommitment, etc.
- **Miner Actor**: ChangeWorkerAddress, WithdrawBalance, GetOwner, GetSectorSize, GetAvailableBalance, etc.
- **Storage Power Actor**: CreateMiner, NetworkRawPower, MinerRawPower, MinerCount
- **Datacap Actor**: Full token interface (Transfer, Balance, Allowance, etc.)
- **Verified Registry Actor**: AddVerifiedClient, GetClaims, ExtendClaimTerms, etc.

## State Tree

Any operation applied (i.e., executed) on the Filecoin VM produces an output in the form of a _State Tree_ (discussed below). The latest _State Tree_ is the current source of truth in the Filecoin Blockchain. The _State Tree_ is identified by a CID, which is stored in the IPLD store.
