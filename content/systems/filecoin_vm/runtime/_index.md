---
title: Runtime
weight: 5
bookCollapseSection: true
dashboardWeight: 1
dashboardState: reliable
dashboardAudit: n/a
dashboardTests: 0
---

# VM Runtime Environment (Inside the VM)

## Receipts

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0049
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0049.md
    description: Added events_root field to support actor events.
  - fip: FIP-0072
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0072.md
    description: Improved event syscall API with separate buffers and refined limits.
-->

A `MessageReceipt` contains the result of a top-level message execution. Every syntactically valid and correctly signed message can be included in a block and will produce a receipt from execution.

A syntactically valid receipt has:

- a non-negative `ExitCode`,
- a non empty `Return` value only if the exit code is zero,
- a non-negative `GasUsed`, and
- an optional `EventsRoot` containing the root CID of an AMT of events emitted during execution (since FIP-0049).

```go
type MessageReceipt struct {
	ExitCode exitcode.ExitCode
	Return   []byte
	GasUsed  int64
	EventsRoot *cid.Cid // Root of AMT<StampedEvent, bitwidth=5> (optional, since FIP-0049)
}
```

## `vm/runtime` Actors Interface

The Actors Interface implementation can be found [here](https://github.com/filecoin-project/specs-actors/blob/master/actors/runtime/runtime.go)

## `vm/runtime` VM Implementation

The Lotus implementation of the Filecoin Virtual Machine runtime can be found [here](https://github.com/filecoin-project/lotus/blob/master/chain/vm/runtime.go)

## Exit Codes

There are some common runtime exit codes that are shared by different actors. Their definition can be found [here](https://github.com/filecoin-project/go-state-types/blob/master/exitcode/common.go).

## Actor Events

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0049
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0049.md
    description: Introduced actor events for external observability.
-->

Since FIP-0049, actors can emit externally observable events during execution. Events are fire-and-forget signals that indicate some relevant circumstance, action, or transition has occurred. They enable external agents to observe on-chain activity without needing to parse state trees or replay messages.

### Event Structure

Events consist of ordered key-value entries with metadata flags:

```rust
/// An event emitted by an actor, stamped with additional metadata by the FVM
struct StampedEvent {
    emitter: u64,        // Actor ID of the emitting actor
    event: ActorEvent,   // The actual event payload
}

/// Event payload as emitted by the actor
type ActorEvent = Vec<Entry>;

struct Entry {
    flags: u64,     // Metadata/hints (e.g., indexing flags)
    key: String,    // UTF-8 key (max 32 bytes)
    codec: u64,     // IPLD codec for the value (currently only IPLD_RAW = 0x55)
    value: Vec<u8>, // Raw value bytes
}
```

### Emitting Events

Actors emit events using the `vm::emit_event` syscall:

```rust
/// Emits an actor event from CBOR-encoded ActorEvent in Wasm memory
fn emit_event(event_off: u32, event_len: u32) -> Result<()>;
```

### Event Accumulation

- Events are accumulated during message execution
- When an actor exits normally (exit code 0), its events are retained
- When an actor exits abnormally (exit code > 0), its events are discarded
- Out of gas or fatal errors discard all events from the call stack

### Event Limits

Since FIP-0072, event limits have been refined:
- Maximum 31 bytes per key (reduced from 32 for compact CBOR encoding)
- Maximum 8 KiB total value size per event
- Maximum 255 entries per event (reduced from 256 for single-byte CBOR length)
- Only IPLD_RAW (0x55) codec currently supported

### Improved Event Syscall API

FIP-0072 introduced an optimized `emit_event` syscall that eliminates CBOR encoding overhead:

**Previous API** (single CBOR-encoded buffer):
```rust
pub fn emit_event(event_off: *const u8, event_len: u32) -> Result<()>
```

**Current API** (three separate buffers):
```rust
pub fn emit_event(
    event_off: *const EventEntry,
    event_len: u32,
    key_off: *const u8,
    key_len: u32,
    value_off: *const u8,
    value_len: u32,
) -> Result<()>
```

Where `EventEntry` is a packed struct:
```rust
#[repr(C, packed)]
pub struct EventEntry {
    pub flags: u64,      // Event flags (indexing hints)
    pub codec: u64,      // Value codec (currently only IPLD_RAW)
    pub key_size: u32,   // Size of key in bytes
    pub value_size: u32, // Size of value in bytes
}
```

This design enables:
- Precise gas charging based on actual data sizes
- Concurrent validation during deserialization
- Elimination of CBOR parsing overhead
- More accurate gas model for non-EVM actors

### Indexing Flags

Events support indexing hints through flags:
- `0x01`: Index by key
- `0x02`: Index by value
- `0x03`: Index by both key and value

Note: Since FIP-0072, indexing costs have been removed from gas calculations as this feature was not being used.
