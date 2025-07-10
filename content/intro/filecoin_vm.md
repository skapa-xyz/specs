---
title: 'Filecoin VM'
weight: 3
dashboardWeight: 0.2
dashboardState: reliable
dashboardAudit: n/a
---

# Filecoin VM

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0030
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0030.md
    description: Introduced the Filecoin Virtual Machine (FVM) with WASM-based execution and user-programmable actors.
-->

The majority of Filecoin's user facing functionality (payments, storage market, power table, etc) is managed through the Filecoin Virtual Machine (Filecoin VM). The network generates a series of blocks, and agrees which 'chain' of blocks is the correct one. Each block contains a series of state transitions called `messages`, and a checkpoint of the current `global state` after the application of those `messages`.

The `global state` here consists of a set of `actors`, each with their own private `state`.

An `actor` is the Filecoin equivalent of Ethereum's smart contracts, it is essentially an 'object' in the filecoin network with state and a set of methods that can be used to interact with it. Every actor has a Filecoin balance attributed to it, a `state` pointer, a `code` CID which tells the system what type of actor it is, and a `nonce` which tracks the number of messages sent by this actor. Since the introduction of the FVM in FIP-0030, actors can be either built-in system actors (like the storage market or miner actors) or user-deployed actors running custom WASM bytecode.

There are two routes to calling a method on an `actor`. First, to call a method as an external participant of the system (aka, a normal user with Filecoin) you must send a signed `message` to the network, and pay a fee to the miner that includes your `message`. The signature on the message must match the key associated with an account with sufficient Filecoin to pay for the message's execution. The fee here is equivalent to transaction fees in Bitcoin and Ethereum, where it is proportional to the work that is done to process the message (Bitcoin prices messages per byte, Ethereum uses the concept of 'gas'. We also use 'gas').

Second, an `actor` may call a method on another actor during the invocation of one of its methods. However, the only time this may happen is as a result of some actor being invoked by an external users message (note: an actor called by a user may call another actor that then calls another actor, as many layers deep as the execution can afford to run for).

For full implementation details, see the [VM Subsystem](systems/filecoin_vm).
