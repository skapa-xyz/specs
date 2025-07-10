---
title: State Tree
weight: 2
dashboardWeight: 1.5
dashboardState: reliable
dashboardAudit: wip
dashboardTests: 0
---

# State Tree

The State Tree is the output of the execution of any operation applied on the Filecoin Blockchain. The on-chain (i.e., VM) state data structure is a map (in the form of a Hash Array Mapped Trie - HAMT v3) that binds addresses to actor states. The current State Tree function is called by the VM upon every actor method invocation.

The State Tree uses the optimized HAMT v3 data structure which provides efficient lookups and updates while minimizing gas costs through dirty node tracking and deferred writes. See the [Data Structures appendix](appendix#hamt-hash-array-mapped-trie) for details on HAMT v3 optimizations.

{{<embed src="https://github.com/filecoin-project/lotus/blob/master/chain/state/statetree.go"  lang="go" symbol="StateTree">}}
