---
title: PoRep
weight: 1
dashboardWeight: 2
dashboardState: reliable
dashboardAudit: wip
dashboardTests: 0
---

# Proof-of-Replication (PoRep)

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0059
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0059.md
    description: Introduced Synthetic PoRep to reduce temporary storage requirements between PreCommit and ProveCommit.
-->

In order to register a sector with the Filecoin network, the sector has to be sealed. Sealing is a computation-heavy process that produces a unique representation of the data in the form of a proof, called **_Proof-of-Replication_** or PoRep.

The PoRep proof ties together: i) the data itself, ii) the miner actor that performs the sealing and iii) the time when the specific data has been sealed by the specific miner. In other words, if the same miner attempts to seal the same data at a later time, then this will result in a different PoRep proof. Time is included as the blockchain height when sealing took place and the corresponding chain reference is called `SealRandomness`.

Once the proof has been generated, the miner runs a SNARK on the proof in order to compress it and submits the result to the blockchain. This constitutes a certification that the miner has indeed replicated a copy of the data they agreed to store.

The PoRep process includes the following two phases:

- **Sealing preCommit phase 1.** In this phase, PoRep SDR [encoding](sdr#encoding) and [replication](sdr#replication) takes place.
- **Sealing preCommit phase 2.** In this phase, [Merkle proof and tree generation](sdr#merkle-proofs) is performed using the Poseidon hashing algorithm.

## Synthetic PoRep

Since FIP-0059, storage providers can optionally use Synthetic PoRep, which significantly reduces the temporary storage requirements between PreCommit and ProveCommit from ~400GiB to ~25GiB, with no impact on security.

### How Synthetic PoRep Works

1. **Challenge Generation**: Based on the CommR, the storage provider generates N_syn = 2^18 synthetic challenges before submitting PreCommit on-chain
2. **Precomputation**: The provider computes vanilla proofs for all synthetic challenges and stores them (~25GiB)
3. **Layer Removal**: Since all possible challenges are precomputed, the provider can delete the ~400GiB of layer data
4. **Interactive Proving**: After PreCommitChallengeDelay (150 epochs), the provider uses on-chain randomness to select N_verified = 176 challenges from the precomputed set
5. **SNARK Generation**: The provider generates SNARK proofs only for the selected challenges

### Benefits

- **Storage Reduction**: >90% reduction in temporary storage requirements
- **Same Security**: No impact on PoRep security guarantees
- **Optional Feature**: Providers can choose between standard and synthetic PoRep
- **No Additional Overhead**: Same proving costs and on-chain flow

### Proof Types

Two new proof types support Synthetic PoRep:
- `RegisteredSealProof_SynthStackedDrg32GiBV1` (32 GiB sectors)
- `RegisteredSealProof_SynthStackedDrg64GiBV1` (64 GiB sectors)
