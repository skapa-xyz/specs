---
title: Storage Market Actor
weight: 1
dashboardWeight: 2
dashboardState: reliable
dashboardAudit: done
dashboardAuditURL: /#section-appendix.audit_reports.actors
dashboardAuditDate: '2020-10-19'
dashboardTests: 0
math-mode: true
---

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0020
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0020.md
    description: WithdrawBalance now returns the actual amount withdrawn.
-->

# Storage Market Actor

The `StorageMarketActor` is responsible for processing and managing on-chain deals. This is also the entry point of all storage deals and data into the system. It maintains a mapping of `StorageDealID` to `StorageDeal` and keeps track of locked balances of `StorageClient` and `StorageProvider`. When a deal is posted on chain through the `StorageMarketActor`, it will first check if both transacting parties have sufficient balances locked up and include the deal on chain.

## `StorageMarketActor` implementation

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/market/market_actor.go" lang="go">}}

## `StorageMarketActorState` implementation

**Storage Market Actor Statuses**
{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/market/market_state.go" lang="go">}}

**Storage Market Actor Balance states and mutations**

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/market/market_balances.go" lang="go">}}

## Storage Deal Collateral

Apart from [Initial Pledge Collateral and Block Reward Collateral](miner_collaterals) discussed earlier, the third form of collateral is provided by the storage provider to _collateralize deals_, is called _Storage Deal Collateral_ and is held in the `StorageMarketActor`.

There is a minimum amount of collateral required by the protocol to provide a minimum level of guarantee, which is agreed upon by the storage provider and client off-chain. However, miners can offer a higher deal collateral to imply a higher level of service and reliability to potential clients. Given the increased stakes, clients may associate additional provider deal collateral beyond the minimum with an increased likelihood that their data will be reliably stored.

Provider deal collateral is only slashed when a sector is terminated before the deal expires. If a miner enters Temporary Fault for a sector and later recovers from it, no deal collateral will be slashed.

This collateral is returned to the storage provider when all deals in the sector successfully conclude. Upon graceful deal expiration, storage providers must wait for finality number of epochs (as defined in [Finality](expected_consensus#finality-in-ec)) before being able to withdraw their `StorageDealCollateral` from the `StorageMarketActor`.

```text
$$MinimumProviderDealCollateral = 1\% \times FILCirculatingSupply \times \frac{DealRawByte}{max(NetworkBaseline, NetworkRawBytePower)}$$
```

## Balance Withdrawals

The Storage Market Actor maintains escrow balances for both clients and providers. These balances can be withdrawn using the `WithdrawBalance` method. As of FIP-0020, this method returns the actual amount withdrawn, which may be less than the requested amount if the available balance is insufficient. This improvement provides better visibility and traceability of FIL flow, particularly important for financial reporting.
