---
title: Storage Miner Actor
weight: 5
dashboardWeight: 2
dashboardState: wip
dashboardAudit: done
dashboardAuditURL: /#section-appendix.audit_reports.actors
dashboardAuditDate: '2020-10-19'
dashboardTests: 0
---

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0020
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0020.md
    description: WithdrawBalance now returns the actual amount withdrawn.
-->

# Storage Miner Actor

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_state.go"  lang="go" symbol="State" title="Storage Miner Actor State">}}

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_actor.go"  lang="go" title="Storage Miner Actor">}}

## Balance Management

Storage miners maintain balances for various purposes including collateral and rewards. The `WithdrawBalance` method allows authorized parties (owner or beneficiary) to withdraw available funds from the miner actor. Following FIP-0020, this method returns the actual amount withdrawn, providing transparency when the available balance is less than the requested withdrawal amount. This enhancement improves financial tracking and reporting for mining operations.
