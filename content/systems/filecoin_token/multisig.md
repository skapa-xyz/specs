---
title: Multisig Wallet
weight: 4
bookCollapseSection: true
dashboardWeight: 1
dashboardState: reliable
dashboardAudit: done
dashboardAuditURL: /#section-appendix.audit_reports.actors
dashboardAuditDate: '2020-10-19'
dashboardTests: 0
---

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0062
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0062.md
    description: Added fallback method handler for method numbers ≥ 2^24 to enable value transfers from EVM actors.
-->

# Multisig Wallet & Actor

The Multisig actor is a single actor representing a group of Signers. Signers may be external users, other Multisigs, or even the Multisig itself. There should be a maximum of 256 signers in a multisig wallet. In case more signers are needed, then the multisigs should be combined into a tree.

The implementation of the Multisig Actor can be found [here](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/multisig/multisig_actor.go).

The Multisig Actor statuses can be found [here](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/multisig/multisig_state.go).

## Method Handling

Since FIP-0062, the multisig actor includes a fallback method handler for method numbers greater than or equal to 2^24 (the first exported method as per FRC-0042). This enables the multisig actor to:

- Receive value transfers from EVM runtime actors (smart contracts), Ethereum accounts, and placeholders
- Accept calls with method numbers ≥ 2^24, which are handled as no-ops returning success
- Maintain compatibility with the Filecoin EVM ecosystem where actors use `MethodNum = FRC-42("InvokeEVM")` for outbound calls

This behavior brings multisig actors in line with account actors, which received similar functionality in FIP-0050. Prior to this change, multisigs would reject such calls, causing value transfers to fail when originating from EVM-based actors.
