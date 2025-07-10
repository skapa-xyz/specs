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
  - fip: FIP-0022
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0022.md
    description: PublishStorageDeals no longer fails entirely when individual deals are invalid.
  - fip: FIP-0060
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0060.md
    description: Increased deal maintenance interval from 1 day to 30 days to reduce cron execution costs.
  - fip: FIP-0074
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0074.md
    description: Removed automatic deal settlement for new deals, added manual SettleDealPayments method.
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

## Deal Publishing

The `PublishStorageDeals` method is used to publish storage deals on-chain. As of FIP-0022, this method has improved error handling that prevents a single invalid deal from causing the entire batch to fail. Instead:

- Valid deals in the batch are successfully published
- Invalid deals (due to validation errors, insufficient balance, or other issues) are dropped
- The return value includes both the deal IDs for successful deals and a bitfield indicating which deals from the input were valid

This change significantly improves the user experience for storage providers by making deal publishing more resilient to individual deal failures. The method only returns an error if all deals fail validation or if an internal error occurs.

## Balance Withdrawals

The Storage Market Actor maintains escrow balances for both clients and providers. These balances can be withdrawn using the `WithdrawBalance` method. As of FIP-0020, this method returns the actual amount withdrawn, which may be less than the requested amount if the available balance is insufficient. This improvement provides better visibility and traceability of FIL flow, particularly important for financial reporting.

## Deal Maintenance and Settlement

### Automatic Settlement (Legacy)

Prior to FIP-0074, the Storage Market Actor performed automatic maintenance on all active deals through Filecoin's cron mechanism every 30 days (FIP-0060). This maintenance included:
- Processing incremental payments from clients to providers
- Handling deal state updates
- Cleaning up expired deals

Since FIP-0074, automatic settlement only applies to deals activated before the upgrade. New deals require manual settlement.

### Manual Deal Settlement

FIP-0074 introduced the `SettleDealPayments` method to allow storage providers to manually settle deal payments:

```rust
struct SettleDealPaymentsParams {
    Deals: Bitfield,  // Deal IDs to settle
}

struct SettleDealPaymentsReturn {
    Results: {
        SuccessCount: u32,
        FailCodes: []{ Index: u32, ExitCode: ExitCode },
    },
    Settlements: []DealSettlementSummary,
}

struct DealSettlementSummary {
    Payment: TokenAmount,  // Incremental amount paid to provider
    Completed: bool,       // Whether deal has settled for final time
}
```

#### Settlement Rules
- **Non-existent deals**: Fail with `USR_NOT_FOUND`
- **Unactivated/unexpired deals**: Succeed with no effect
- **Expired or completed deals**: Fail with `EX_DEAL_EXPIRED`
- **Terminated deals**: Abort with `USR_ILLEGAL_ARGUMENT`

### Termination Processing

Since FIP-0074, the `OnMinerSectorsTerminate` method immediately:
1. Makes final payment settlement
2. Charges applicable penalties
3. Cleans up deal state

This replaces the previous deferred cleanup via cron, ensuring immediate state consistency.

### Impact on Gas Costs

The shift from automatic to manual settlement:
- **Removes risk**: Cron operations no longer threaten to exceed chain processing capacity
- **Aligns costs**: Storage providers pay gas for settlement, incentivizing efficient batching
- **Enables scaling**: Deal volume can grow without impacting cron execution
- **Levels playing field**: User-programmed markets aren't disadvantaged by subsidized settlement

Storage providers can optimize costs by:
- Settling multiple deals in a single transaction
- Choosing periods of low gas demand
- Settling only payment-bearing deals (zero-fee deals can remain unsettled)
