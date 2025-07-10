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
  - fip: FIP-0029
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0029.md
    description: Added beneficiary address for financial control separation from owner.
-->

# Storage Miner Actor

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_state.go"  lang="go" symbol="State" title="Storage Miner Actor State">}}

{{<embed src="https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_actor.go"  lang="go" title="Storage Miner Actor">}}

## Balance Management

Storage miners maintain balances for various purposes including collateral and rewards. The `WithdrawBalance` method allows authorized parties to withdraw available funds from the miner actor. 

### Beneficiary Address

Since FIP-0029, storage miners can designate a beneficiary address that takes over financial control from the owner. This separation enables more flexible financial arrangements such as lending markets and improved security. Key features include:

- **Beneficiary Role**: The beneficiary address receives all withdrawn funds, even when withdrawals are initiated by the owner
- **Quota and Expiration**: Beneficiaries have a quota (maximum withdrawal amount) and expiration date defined in the BeneficiaryTerm
- **Change Process**: Changing the beneficiary requires approval from the owner, current beneficiary, and proposed beneficiary (with auto-approval in certain cases)
- **Default Behavior**: New miners without a specified beneficiary have their beneficiary set to the owner address for backward compatibility

### Withdrawal Process

The `WithdrawBalance` method can be called by either the owner or beneficiary address, but funds are always sent to the beneficiary. Following FIP-0020, this method returns the actual amount withdrawn, providing transparency when the available balance is less than the requested withdrawal amount. 

If the beneficiary term has expired or the quota is exhausted, withdrawal attempts will fail with a USR_FORBIDDEN error until the beneficiary is updated. This differs from owner-to-self withdrawals, which succeed with zero withdrawal when no funds are available.

### Methods

- **WithdrawBalance**: Withdraws available funds to the beneficiary address
- **ChangeBeneficiary**: Proposes or confirms a change to the beneficiary address and/or terms
- **GetBeneficiary**: Retrieves current and proposed beneficiary information
