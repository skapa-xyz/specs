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
  - fip: FIP-0071
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0071.md
    description: Introduced deterministic state access rules to ensure actors can only read state reachable from their state-tree.
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

## Filecoin EVM (FEVM)

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0054
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0054.md
    description: Introduced the Filecoin EVM runtime actor for running Ethereum smart contracts.
-->

Since FIP-0054, Filecoin supports the execution of Ethereum smart contracts through the Filecoin EVM (FEVM) runtime actor. This built-in actor enables Ethereum compatibility while maintaining integration with Filecoin's unique features.

### EVM Runtime Actor

The FEVM runtime actor:
- **Runs EVM bytecode**: Compatible with Ethereum Paris fork plus EIP-3855 (PUSH0 opcode)
- **Embeds an EVM interpreter**: Executes smart contracts within the FVM environment
- **Translates opcodes**: Maps Ethereum operations to Filecoin primitives
- **Manages state**: Maps EVM storage model to Filecoin's IPLD-based state

### Key Features

1. **Address Mapping**: 
   - Ethereum addresses are mapped to Filecoin f4 addresses (via Ethereum Address Manager)
   - Maintains compatibility with existing Ethereum tooling
   - Supports CREATE and CREATE2 deployment patterns

2. **Precompiles Support**:
   - All standard Ethereum precompiles (ecrecover, SHA256, etc.)
   - Filecoin-specific precompiles for native actor interaction
   - Call actor methods, resolve addresses, and access Filecoin state

3. **State Management**:
   - EVM storage is persisted as IPLD blocks
   - Efficient storage through deduplication and content addressing
   - Compatible with Ethereum's storage slot model

### Differences from Ethereum

While striving for maximum compatibility, some differences exist:
- **Block Time**: ~30 seconds vs Ethereum's ~12 seconds
- **Chain ID**: Filecoin mainnet uses 314, Calibration testnet uses 314159
- **Gas Model**: Different pricing due to FVM's execution model
- **No Pending Pool**: Transactions execute in the epoch they're included

### Actor Interface

The EVM runtime actor exposes these main methods:
- **Constructor**: Deploys new EVM contracts
- **InvokeContract**: Executes contract methods
- **GetBytecode**: Retrieves deployed bytecode
- **GetStorageAt**: Reads contract storage

## State Tree

Any operation applied (i.e., executed) on the Filecoin VM produces an output in the form of a _State Tree_ (discussed below). The latest _State Tree_ is the current source of truth in the Filecoin Blockchain. The _State Tree_ is identified by a CID, which is stored in the IPLD store.

## Deterministic State Access

Since FIP-0071, the FVM enforces deterministic rules for state access to ensure network consensus and prepare for user-defined WebAssembly actors. These rules guarantee that actors can only access state that is explicitly "reachable" from their execution context.

### Reachable Set

The FVM maintains a "reachable set" of IPLD blocks (identified by CIDs) that an actor instance can access. This set is per-actor-invocation and starts with:

1. **Actor's state root**: The CID of the actor's state tree
2. **Message parameters**: Any IPLD blocks passed as parameters from other actors
3. **Return values**: Blocks returned from calls to other actors

State is considered "reachable" if it can be accessed by traversing IPLD links (CIDs) from these roots.

### IPLD State Access Rules

Actors interact with state through IPLD syscalls, which enforce the following rules:

#### Reading State (`ipld::block_open`)
- Actors can only open blocks whose CIDs are in the reachable set
- When a block is opened, all CIDs it references are added to the reachable set
- Gas is charged for tracking reachable CIDs (`ipld_link_tracked`: 550 gas per CID)

#### Writing State (`ipld::block_create`)
- New blocks can only reference CIDs currently in the reachable set
- The FVM performs link analysis to extract all CIDs from the block
- Gas is charged for checking CID reachability (`ipld_link_checked`: 500 gas per CID)

#### State Root Updates (`self::set_root`)
- Actors can only set their state root to a CID in the reachable set
- The root must be a blake2b-256 CID, not an identity-hashed inline block

### Link Analysis

The FVM performs IPLD link analysis to determine which blocks a given block references:

**Supported Codecs**:
- Raw (0x55): Contains no links
- CBOR (0x51): Contains no links
- DagCBOR (0x71): Can contain links to other IPLD blocks

**Allowed CIDs**:
- Blake2b-256 hashes (32 bytes)
- Identity hashes (up to 64 bytes, inlining the block)
- Codecs must be in the supported set

**Gas Charges**:
- `ipld_cbor_scan_per_field`: 85 gas per CBOR field parsed
- `ipld_cbor_scan_per_cid`: 950 gas per CID encountered

### Cross-Actor Communication

When actors communicate via `send::send`:
1. IPLD blocks are passed by handle, not by CID
2. The receiving actor gets a copy of the block in its block table
3. All CIDs reachable from the transferred block are added to the receiver's reachable set
4. This ensures actors can share state while maintaining isolation

### Security Benefits

These rules prevent several potential issues:
- **Consensus forks**: Actors cannot read arbitrary blocks that might exist in some nodes but not others
- **State pollution**: Actors cannot reference garbage or leftover state from previous tipsets
- **Deterministic execution**: All state access is predictable and reproducible across the network

This deterministic state access model is essential for supporting arbitrary user-defined actors while maintaining network security and consensus.
