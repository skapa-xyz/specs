---
title: Adding Storage
weight: 7
dashboardWeight: 2
dashboardState: stable
dashboardAudit: wip
dashboardTests: 0
---

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0076
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0076.md
    description: Added direct data onboarding methods ProveCommitSectors3 and ProveReplicaUpdates3 that bypass built-in market deals.
-->

# Adding Storage

A Miner adds more storage in the form of Sectors. Adding more storage is a two-step process:

1. **PreCommitting a Sector**: A Miner publishes a Sector's SealedCID and data commitment (unsealed CID), through `miner.PreCommitSectorBatch2`, and makes a deposit. The Sector is now registered to the Miner, and the Miner must ProveCommit the Sector or lose their deposit.
2. **ProveCommitting a Sector**: The Miner provides a Proof of Replication (PoRep) for the Sector through one of several methods:
   - `miner.ProveCommitSector` or `miner.ProveCommitAggregate` - Traditional methods requiring built-in market deals
   - `miner.ProveCommitSectors3` - Direct data onboarding method that bypasses the built-in market actor
   
   This proof must be submitted AFTER a delay (the InteractiveEpoch), and BEFORE PreCommit expiration.

This two-step process provides assurance that the Miner's PoRep _actually proves_ that the Miner has replicated the Sector data and is generating proofs from it:

- ProveCommitments must happen AFTER the InteractiveEpoch (150 blocks after Sector PreCommit), as the randomness included at that epoch is used in the PoRep.
- ProveCommitments must happen BEFORE the PreCommit expiration, which is a boundary established to make sure Miners don't have enough time to "fake" PoRep generation.

For each Sector successfully ProveCommitted, the Miner becomes responsible for continuously proving the existence of their Sectors' data. In return, the Miner is awarded storage power.

# Upgrading Sectors

Miners are granted storage power in exchange for the storage space they dedicate to Filecoin. Ideally, this storage space is used to store data on behalf of Clients, but there may not always be enough Clients to utilize all the space a Miner has to offer.

In order for a Miner to maximize storage power (and profit), they should take advantage of all available storage space immediately, _even before they find enough Clients to use this space_.

To facilitate this, there are _two types_ of Sectors that may be sealed and ProveCommitted:

- **Regular Sector**: A Sector that contains Client data
- **Committed Capacity (CC) Sector**: A Sector with no data (all zeroes)

Miners are free to choose which types of Sectors to store. CC sectors, in particular, allow Miners to immediately make use of existing disk space: earning storage power and a higher chance at producing a block. Miners can decide if they should upgrade their CC sectors to take client deals or continue proving CC sectors. Currently, CC sectors store randomness by default in client implementation, but this does not preclude miners from storing any type of useful data that increase their private utility in CC sectors (as long as it is legal). The protocol expects that new use-cases and diversity will emerge out of such behaviour.

To incentivize Miners to hoard storage space and dedicate it to Filecoin, CC Sectors have a unique capability: **they can be "upgraded" to Regular Sectors** (also called "replacing a CC Sector").

Miners upgrade their ProveCommitted CC Sectors by PreCommitting a Regular Sector, and specifying that it should replace an existing CC Sector. Once the Regular Sector is successfully ProveCommitted, it will replace the existing CC Sector. If the newly ProveCommitted Regular sector contains a Filecoin Plus deal, i.e., a deal with higher Sector Quality, then the miner's storage power will increase accordingly.

Upgrading capacity currently involves resealing, that is, creating a unique representation of the new data included in the Sector through a computationally intensive process. Looking ahead, committed capacity upgrades should eventually be possible without a reseal. A succinct and publicly verifiable proof that the committed capacity has been correctly replaced with replicated data should achieve this goal. However, this mechanism must be fully specified to preserve the security and incentives of the network before it can be implemented and is, therefore, left as a future improvement.

## Direct Data Onboarding

Direct data onboarding allows storage providers to commit data to sectors without requiring built-in market deals. This significantly reduces gas costs for common use cases like verified deals with no on-chain payments.

### ProveCommitSectors3

The `ProveCommitSectors3` method (method 34) enables direct sector activation with the following features:

- **Piece Manifests**: Explicitly declare all pieces of data in each sector
- **Verified Allocations**: Claim DataCap allocations directly from the verified registry
- **Actor Notifications**: Notify actors (currently only built-in market) when sectors are activated
- **Batch Support**: Process multiple sectors with either individual or aggregate proofs

Key parameters:
- `SectorActivations`: Manifests describing data in each sector
- `SectorProofs` or `AggregateProof`: Proof(s) of replication
- `RequireActivationSuccess`: Whether to abort if any sector fails
- `RequireNotificationSuccess`: Whether to abort if any notification fails

### ProveReplicaUpdates3

The `ProveReplicaUpdates3` method (method 35) provides similar functionality for updating existing sectors:

- Updates sectors with new data while maintaining the same sector number
- Supports the same piece manifest and notification features as `ProveCommitSectors3`
- Enables capacity sectors to be updated with real data

### Direct Onboarding Workflow

1. **For verified data**: Client allocates DataCap directly with verified registry
2. **PreCommit**: Storage provider specifies unsealed CID (CommD) but no deal IDs
3. **ProveCommit**: Storage provider uses `ProveCommitSectors3` with:
   - Piece manifests declaring sector contents
   - Verified allocation IDs to claim (if applicable)
   - Actor addresses to notify (if applicable)

### Benefits

- **Reduced Gas Costs**: Bypass built-in market actor when deals aren't needed
- **Flexibility**: Support for future user-programmed storage applications
- **Efficiency**: Batch operations and optimized gas usage
- **Simplicity**: Direct path for common Filecoin Plus deals

### Deprecated Methods

The following methods are removed as of FIP-0076:
- `PreCommitSector` (method 6)
- `PreCommitSectorBatch` (method 25)
- `ProveReplicaUpdates2` (method 29)

Storage providers should use `PreCommitSectorBatch2` for pre-commitment and the new `ProveCommitSectors3` or `ProveReplicaUpdates3` for direct onboarding.
