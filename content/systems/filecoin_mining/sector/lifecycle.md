---
title: Sector Lifecycle
weight: 2
dashboardWeight: 2
dashboardState: stable
dashboardAudit: n/a
dashboardTests: 0
---

# Sector Lifecycle

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0008
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0008.md
    description: Added PreCommitSectorBatch method to enable batch pre-commitment of up to 256 sectors.
  - fip: FIP-0013
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0013.md
    description: Added ProveCommitSectorAggregated method to enable aggregated proof verification for multiple sectors.
  - fip: FIP-0014
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0014.md
    description: Allowed V1 proof sectors to be extended up to a maximum of 540 days.
-->

Once the sector has been generated and the deal has been incorporated into the Filecoin blockchain, the storage miner begins generating Proofs-of-Spacetime (PoSt) on the sector, starting to potentially win block rewards and also earn storage fees. Parameters are set so that miners generate and capture more value if they guarantee that their sectors will be around for the duration of the original contract. However, some bounds are placed on a sectorʼs lifetime to improve the network performance.

In particular, as sectors of shorter lifetime are added, the networkʼs capacity can be bottlenecked. The reason is that the chainʼs bandwidth is consumed with new sectors only replacing expiring ones. As a result, a minimum sector lifetime of six months was introduced to more effectively utilize chain bandwidth and miners have the incentive to commit to sectors of longer lifetime. The maximum sector lifetime is limited by the security of the present proofs construction. For a given set of proofs and parameters, the security of Filecoinʼs Proof-of-Replication (PoRep) is expected to decrease as sector lifetimes increase.

It is reasonable to assume that miners enter the network by adding Committed Capacity sectors, that is, sectors that do not contain user data. Once miners agree storage deals with clients, they upgrade their sectors to Regular Sectors. Alternatively, if they find Filecoin Plus clients and agree a storage deal with them, they upgrade their sector accordingly. Depending on whether or not a sector includes a Filecoin Plus deal, the miner acquires the corresponding storage power in the network.

All sectors are expected to remain live until the end of their sector lifetime and early dropping of sectors will result in slashing. This is done to provide clients a certain level of guarantee on the reliability of their hosted data. Sector termination comes with a corresponding _termination fee_.

As with every system it is expected that sectors will present faults. Although this might degrade the quality offered by the network, the reaction of the miner to the fault drives system decisions on whether or not the miner should be penalized. A miner can recover the faulty sector, let the system terminate the sector automatically after 42 days of faults, or proactively terminate the sector immediately in the case of unrecoverable data loss. In case of a faulty sector, a small penalty fee approximately equal to the block reward that the sector would win per day is applied. The fee is calculated per day of the sector being unavailable to the network, i.e. until the sector is recovered or terminated.

Miners can extend the lifetime of a sector at any time, though the sector will be expected to remain live until it has reached the end of the new sector lifetime. This can be done by submitting a `ExtendedSectorExpiration` message to the chain.

## Sector Extension Limitations

### V1 Proof Sectors
Sectors sealed using V1 proof before network version 7 (November 27, 2020) have a special limitation on their maximum lifetime. These sectors can only be extended up to a maximum total lifetime of 540 days, including the days they have already been active. This restriction was introduced to address potential long-term security concerns with the V1 proof construction while still allowing miners who sealed these sectors to benefit from extensions.

A sector can be in one of the following states.

| State          | Description                                                                                                                                           |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Precommitted` | Miner seals sector and submits `miner.PreCommitSector` or `miner.PreCommitSectorBatch` (up to 256 sectors per batch)                                  |
| `Committed`    | Miner generates a Seal proof and submits `miner.ProveCommitSector` or `miner.ProveCommitSectorAggregated`                                             |
| `Active`       | Miner generate valid PoSt proofs and timely submits `miner.SubmitWindowedPoSt`                                                                        |
| `Faulty`       | Miner fails to generate a proof (see Fault section)                                                                                                   |
| `Recovering`   | Miner declared a faulty sector via `miner.DeclareFaultRecovered`                                                                                      |
| `Terminated`   | Either sector is expired, or early terminated by a miner via `miner.TerminateSectors`, or was failed to be proven for 42 consecutive proving periods. |

## Batch Operations

To improve gas efficiency and reduce chain congestion, miners can use batch methods for sector operations:

### PreCommitSectorBatch
The `PreCommitSectorBatch` method allows miners to pre-commit up to 256 sectors in a single transaction. This method provides significant gas savings by:
- Fetching reward and power statistics only once for the entire batch
- Allocating sector numbers in batch rather than individually
- Loading and storing state structures (HAMT, AMT) only once
- Invoking market actor verification once for all sectors

High-growth miners benefit most from batching, as it amortizes per-sector costs across multiple sectors. The 256 sector limit per batch supports up to 8 EiB of 32 GiB sectors per year for a single miner.

### ProveCommitSectorAggregated
The `ProveCommitSectorAggregated` method allows miners to prove-commit multiple sectors at once using aggregated proofs. This method provides significant gas savings by:
- Using aggregated proof verification that scales logarithmically with the number of sectors
- Amortizing state access costs across multiple prove commits
- Batching market actor `ComputeDataCommitment` calls
- Eliminating the need for temporary storage and cron-batching used in individual prove commits

The method supports a minimum of 4 and a maximum of 819 sectors per aggregation. The aggregated proof uses novel cryptographic techniques to drastically reduce per-sector proof size and verification times. The maximum delay between pre-commit and prove-commit is extended to 30 days plus PreCommitChallengeDelay to allow miners of all sizes to accumulate enough sectors for efficient aggregation.
