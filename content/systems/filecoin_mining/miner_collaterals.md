---
title: Miner Collaterals
weight: 3
bookCollapseSection: false
dashboardWeight: 2
dashboardState: reliable
dashboardAudit: n/a
dashboardTests: 0
---

# Miner Collaterals

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0034
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0034.md
    description: Fixed pre-commit deposit to sector quality 10 value regardless of sector content.
-->

Most permissionless blockchain networks require upfront investment in resources in order to participate in the consensus. The more power an entity has on the network, the greater the share of total resources it needs to own, both in terms of physical resources and/or staked tokens (collateral).

Filecoin must achieve security via the dedication of resources. By design, Filecoin mining requires commercial hardware only (as opposed to ASIC hardware) that is cheap in amortized cost and easy to repurpose, which means the protocol cannot solely rely on the hardware as the capital investment at stake for attackers. Filecoin also uses upfront token collaterals, as in proof-of-stake protocols, proportional to the storage hardware committed. This gets the best of both worlds: attacking the network requires both acquiring and running the hardware, but it also requires acquiring large quantities of the token.

To satisfy the multiple needs for collateral in a way that is minimally burdensome to miners, Filecoin includes three different collateral mechanisms: _initial pledge collateral, block reward as collateral, and storage deal provider collateral_. The first is an initial commitment of filecoin that a miner must provide with each sector. The second is a mechanism to reduce the initial token commitment by vesting block rewards over time. The third aligns incentives between miner and client, and can allow miners to differentiate themselves in the market. The remainder of this subsection describes each in more detail.

## Pre-Commit Deposit

Before a sector can be proven and activated, storage providers must submit a pre-commit deposit (PCD) that serves as an early commitment to the sector. Since FIP-0034, this deposit is set to a fixed value regardless of sector content, calculated as the 20-day projection of expected reward for a sector with quality 10 (i.e., completely filled with verified deals).

This fixed pre-commit deposit:
- Simplifies the onboarding process by removing the dependency on deal content
- Provides adequate security against proof-of-replication cheating attempts
- Enables future programmable storage markets by decoupling sector onboarding from the built-in market actor
- Ensures the deposit is always less than the initial pledge, minimizing additional capital requirements

The pre-commit deposit is forfeited if the sector fails to be proven within the required timeframe, incentivizing providers to complete the sector onboarding process.

## Initial Pledge Collateral

Filecoin Miners must commit resources in order to participate in the economy; the protocol can use the minersʼ stake in the network to ensure that rational behavior benefits the network, rewarding the creation of value and penalizing malicious behavior via slashing. The pledge size is meant to adequately incentivize the fulfillment of a sectorʼs promised lifetime and provide sufficient consensus security.

Hence, the initial pledge function consists of two components: a _storage pledge_ and a _consensus pledge_.

{{<katex>}}

$SectorInitialPledge = SectorInitialStoragePledge + SectorInitialConsensusPledge$

{{</katex>}}

The storage pledge protects the networkʼs quality-of-service for clients by providing starting collateral for the sector in the event of slashing. The storage pledge must be small enough to be feasible for miners joining the network, and large enough to collateralize storage against early faults, penalties, and fees. The vesting of block rewards and the use of unvested rewards as additional collateral reduces initial storage pledge without compromising the incentive alignment of the network. This is discussed in more depth in the following subsection. A balance is achieved by using an initial storage pledge amount approximately sufficient to cover 7 days worth of Sector fault fee and 1 Sector fault detection fee. This is denominated in the number of days of future rewards that a sector is expected to earn.

{{<katex>}}

$SectorInitialStoragePledge = Estimated20DaysSectorBlockReward$

{{</katex>}}

Since the storage pledge per sector is based on the expected block reward that sector will win, the storage pledge is independent of the networkʼs total storage. As a result, the total network storage pledge depends solely on future block reward. Thus, while the storage pledge provides a clean way to reason about the rationality of adding a sector, it does not provide sufficient long-term security guarantees to the network, making consensus takeovers less costly as the block reward decreases. As such, the second half of the initial pledge function, the consensus pledge, depends on both the amount of quality-adjusted power (QAP) added by the sector and the network circulating supply. The network targets approximately 30% of the network's circulating supply locked up in initial consensus pledge when it is at or above the baseline. This is achieved with a small pledge share allocated to sectors based on their share of the networkʼs quality-adjusted power. Given an exponentially growing baseline, initial pledge per unit QAP should decrease over time, as should other mining costs.

{{<katex>}}

$SectorInitialConsensusPledge = 30\% \times FILCirculatingSupply \times \frac{SectorQAP}{max(NetworkBaseline, NetworkQAP)}$

{{</katex>}}

## Block Reward Collateral

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0004
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0004.md
    description: Introduced 25% immediate liquidity for block rewards, with 75% vesting over 180 days.
  - fip: FIP-0005
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0005.md
    description: Removed vesting processing from PreCommitSector and ConfirmSectorProofsValid, leaving it to deadline cron and WithdrawBalance calls.
-->

Clients need reliable storage. Under certain circumstances, miners might agree to a storage deal, then want to abandon it later as a result of increased costs or other market dynamics. A system where storage miners can freely or cheaply abandon files would drive clients away from Filecoin as a result of serious data loss and low quality of service. To make sure all the incentives are correctly aligned, Filecoin penalizes miners that fail to store files for the promised duration. As such, high collateral could be used to incentivize good behavior and improve the networkʼs quality of service. On the other hand, however, high collateral creates barriers to miners joining the network. Filecoin's constructions have been designed such that they hit the right balance.

In order to reduce the upfront collateral that a miner needs to provide, the block reward is used as collateral. This allows the protocol to require a smaller but still meaningful initial pledge. Block rewards earned by a sector are subject to slashing if a sector is terminated before its expiration. However, due to chain state limitations, the protocol is unable to do accounting on a per sector level, which would be the most fair and accurate. Instead, the chain performs a per-miner level approximation. Sublinear vesting provides a strong guarantee that miners will always have the incentive to keep data stored until the deal expires and not earlier. An extreme vesting schedule would release all tokens that a sector earns only when the sector promise is fulfilled.

However, the protocol should provide liquidity for miners to support their mining operations, and releasing rewards all at once creates supply impulses to the network. Moreover, there should not be a disincentive for longer sector lifetime if the vesting duration also depends on the lifetime of the sector. As a result, the protocol implements a balanced approach:
- 25% of block rewards are immediately available for withdrawal, providing liquidity for mining operations
- 75% of block rewards vest linearly over 180 days and serve as collateral

This structure creates the necessary sub-linearity while ensuring miners have access to funds for operational needs. The vested portion acts as collateral and is added to pledgeDeltaTotal, aligning long-term incentives.

Vesting is processed through the miner's deadline cron handler every 30 minutes and when WithdrawBalance is called. The vesting table is quantized to 12-hour increments. Miners can manually trigger vesting processing by calling WithdrawBalance with an amount of zero using the miner's Owner address.

In general, fault fees are slashed first from the soonest-to-vest unvested block rewards followed by the minerʼs account balance. When a minerʼs balance is insufficient to cover their minimum requirements, their ability to participate in consensus, win block rewards, and grow storage power will be restricted until their balance is restored. Overall, this reduces the initial pledge requirement and creates a sufficient economic deterrent for faults without slashing the miner's balance for every penalty.

## Storage Deal Collateral

The third form of collateral is provided by the storage provider to collateralize deals. See the [Storage Market Actor](storage_market_actor) for further details on the Storage Deal Collateral.
