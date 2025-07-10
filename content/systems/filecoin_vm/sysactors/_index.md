---
title: System Actors
weight: 6
bookCollapseSection: true
dashboardWeight: 2
dashboardState: reliable
dashboardAudit: done
dashboardAuditURL: /#section-appendix.audit_reports.actors
dashboardAuditDate: '2020-10-19'
dashboardTests: 0
---

# System Actors

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0031
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0031.md
    description: Added state to SystemActor to maintain registry of built-in actor Code CIDs.
  - fip: FIP-0044
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0044.md
    description: Added AuthenticateMessage method to AccountActor for standard authentication.
  - fip: FIP-0054
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0054.md
    description: Added EVM runtime actor for executing Ethereum smart contracts.
  - fip: FIP-0055
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0055.md
    description: Added Ethereum Address Manager and Ethereum Account actors.
-->

There are fourteen (14) builtin System Actors in total, but not all of them interact with the VM. Each actor is identified by a _Code ID_ (or CID).

There are four (4) system actors required for VM processing:

- the [InitActor](sysactors#initactor), which initializes new actors and records the network name, and
- the [CronActor](sysactors#cronactor), a scheduler actor that runs critical functions at every epoch.

There are another three actors that interact with the VM:

- the [AccountActor](sysactors#accountactor) responsible for user accounts (non-singleton), and
- the [RewardActor](sysactors#rewardactor) for block reward and token vesting (singleton).
- the `EthereumAccountActor` responsible for Ethereum EOA accounts, supporting native Ethereum transactions (non-singleton).

The remaining nine (9) builtin System Actors that do not interact directly with the VM are the following:

- `StorageMarketActor`: responsible for managing storage and retrieval deals [[Market Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/market/market_actor.go)]
- `StorageMinerActor`: actor responsible to deal with storage mining operations and collect proofs [[Storage Miner Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_actor.go)]
- `MultisigActor` (or Multi-Signature Wallet Actor): responsible for dealing with operations involving the Filecoin wallet. Since FIP-0062, includes a fallback handler for method numbers ≥ 2^24 to accept value transfers from EVM actors [[Multisig Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/multisig/multisig_actor.go)]
- `PaymentChannelActor`: responsible for setting up and settling funds related to payment channels [[Paych Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/paych/paych_actor.go)]
- `StoragePowerActor`: responsible for keeping track of the storage power allocated at each storage miner [[Storage Power Actor](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/power/power_actor.go)]
- `VerifiedRegistryActor`: responsible for managing the Filecoin Plus program, including Root Key Holders (via multisig), Notaries, Filecoin Plus clients, and DataCap allocations. This actor enables the social trust layer that allows verified data to receive a 10x quality multiplier. Since FIP-0028, it also supports removing DataCap from client addresses through the `RemoveVerifiedClientDatacap` method [[Verifreg Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/verifreg/verified_registry_actor.go)]
- `SystemActor`: general system actor that, since FIP-0031, maintains a registry of built-in actor Code CIDs [[System Actor Repo](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/system/system_actor.go)]
- `EVMRuntimeActor`: responsible for executing Ethereum smart contracts within the Filecoin Virtual Machine. Since FIP-0054, this actor enables EVM compatibility by running EVM bytecode and managing contract state [[EVM Actor Repo](https://github.com/filecoin-project/builtin-actors/tree/master/actors/evm)]
- `EthereumAddressManagerActor` (EAM): singleton actor at f010 that manages the f410 address space for Ethereum addresses. Since FIP-0055, it acts as a factory for creating EVM contracts and Ethereum accounts [[EAM Actor Repo](https://github.com/filecoin-project/builtin-actors/tree/master/actors/eam)]

## CronActor

Built in to the genesis state, the `CronActor`'s dispatch table invokes the `StoragePowerActor` and `StorageMarketActor` for them to maintain internal state and process deferred events. It could in principle invoke other actors after a network upgrade.

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/cron/cron_actor.go"  lang="go">}}

## InitActor

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0048
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0048.md
    description: Added Exec4 method for creating actors with f4 addresses.
-->

The `InitActor` has the power to create new actors, e.g., those that enter the system. It maintains a table resolving a public key and temporary actor addresses to their canonical ID-addresses. Invalid CIDs should not get committed to the state tree.

Since FIP-0048, the InitActor also supports the `Exec4` method, which allows address managers to create new actors with specific f4 addresses. The Exec4 method:
- Computes the f4 address as `4{leb128(caller-actor-id)}{subaddress}`
- Creates a new actor with both an f2 (stable) address and the specified f4 address
- Stores both address mappings in the InitActor's address map
- Is currently restricted to "blessed" address managers

Note that the canonical ID address does not persist in case of chain re-organization. The actor address or public key survives chain re-organization.

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/init/init_actor.go" lang="go">}}

## RewardActor

The `RewardActor` is where unminted Filecoin tokens are kept. The actor distributes rewards directly to miner actors, where they are locked for vesting. The reward value used for the current epoch is updated at the end of an epoch through a cron tick.

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/reward/reward_actor.go"  lang="go">}}

## AccountActor

The `AccountActor` is responsible for user accounts. Account actors are not created by the `InitActor`, but their constructor is called by the system. Account actors are created by sending a message to a public-key style address. The address must be `BLS` or `SECP`, or otherwise there should be an exit error. The account actor is updating the state tree with the new actor address.

Since FIP-0044, the AccountActor implements the `AuthenticateMessage` method, which provides a standard way for actors to authenticate data. This method validates that a given message has been properly authorized by the account through signature verification. This standard authentication interface enables:
- Other actors to verify account authorization without directly handling signatures
- A template for other actors (built-in and user-defined) to implement authentication
- Storage Market and Payment Channel actors to authenticate participants uniformly

The AuthenticateMessage method accepts authorization data (typically a signature) and a message, returning true if the authentication is valid.

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/account/account_actor.go" lang="go" >}}

## EthereumAccountActor

Since FIP-0055, the `EthereumAccountActor` represents Ethereum Externally-Owned Accounts (EOAs) backed by secp256k1 keys. This actor enables native Ethereum transaction support in Filecoin:

- **Ethereum Compatibility**: Accepts native EIP-1559 Ethereum transactions with secp256k1 ECDSA signatures
- **Delegated Signatures**: Uses a new Delegated signature type that carries signatures verified by actor code
- **Address Management**: Associated with f410 addresses managed by the Ethereum Address Manager
- **Universal Methods**: Accepts all methods ≥ 2^24 (FRC-0042 minimum), preparing for future Account Abstraction

The Ethereum Account actor serves as a bridge between Ethereum wallets and the Filecoin network, allowing existing Ethereum tools to interact seamlessly with Filecoin.
