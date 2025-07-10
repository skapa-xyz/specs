---
title: Data Structures
weight: 3
dashboardWeight: 0.2
dashboardState: reliable
dashboardAudit: n/a
---

# Data Structures

## RLE+ Bitset Encoding

RLE+ is a lossless compression format based on [RLE](https://en.wikipedia.org/wiki/Run-length_encoding).
Its primary goal is to reduce the size in the case of many individual bits, where RLE breaks down quickly,
while keeping the same level of compression for large sets of contiugous bits.

In tests it has shown to be more compact than RLE itself, as well as [Concise](https://arxiv.org/pdf/1004.0403.pdf) and [Roaring](https://roaringbitmap.org/).

### Format

The format consists of a header, followed by a series of blocks, of which there are three different types.

The format can be expressed as the following [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form) grammar.

```bnf
    <encoding> ::= <header> <blocks>
      <header> ::= <version> <bit>
     <version> ::= "00"
      <blocks> ::= <block> <blocks> | ""
       <block> ::= <block_single> | <block_short> | <block_long>
<block_single> ::= "1"
 <block_short> ::= "01" <bit> <bit> <bit> <bit>
  <block_long> ::= "00" <unsigned_varint>
         <bit> ::= "0" | "1"
```

An `<unsigned_varint>` is defined as specified [here](https://github.com/multiformats/unsigned-varint).

#### Blocks

The blocks represent how many bits, of the current bit type there are. As `0` and `1` alternate in a bit vector
the inital bit, which is stored in the header, is enough to determine if a length is currently referencing
a set of `0`s, or `1`s.

##### Block Single

If the running length of the current bit is only `1`, it is encoded as a single set bit.

##### Block Short

If the running length is less than `16`, it can be encoded into up to four bits, which a short block
represents. The length is encoded into a 4 bits, and prefixed with `01`, to indicate a short block.

##### Block Long

If the running length is `16` or larger, it is encoded into a varint, and then prefixed with `00` to indicate
a long block.

> **Note:** The encoding is unique, so no matter which algorithm for encoding is used, it should produce
> the same encoding, given the same input.

##### Bit Numbering

For Filecoin, byte arrays representing RLE+ bitstreams are encoded using [LSB 0](https://en.wikipedia.org/wiki/Bit_numbering#LSB_0_bit_numbering) bit numbering.

## HAMT (Hash Array Mapped Trie)

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0007
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0007.md
    description: Upgraded to HAMT v3 with performance optimizations including dirty node tracking, eliminated unnecessary cache clearing, and more efficient pointer serialization.
-->

Filecoin uses HAMT (Hash Array Mapped Trie) v3 as the primary data structure for the global state tree and throughout actor code. The HAMT provides an efficient key-value store with the following characteristics:

### Key Features
- **Efficient lookups and updates**: O(log n) time complexity for get/set operations
- **Space-efficient**: Only allocates nodes as needed
- **Merkle proof compatible**: Each node has a CID for verification

### Version 3 Optimizations
The current HAMT implementation includes several performance optimizations:

1. **Dirty node tracking**: Only flushes modified nodes to the blockstore, reducing unnecessary writes
2. **Persistent caching**: Retains loaded nodes in memory after flush operations
3. **Deferred writes**: All blockstore writes occur during flush rather than immediately
4. **Efficient pointer serialization**: Uses CBOR type discrimination instead of keyed maps, saving 3 bytes per pointer

For implementation details, see the [IPLD hash map spec](https://github.com/ipld/specs/blob/master/data-structures/hashmap.md) and the [go-hamt-ipld](https://github.com/filecoin-project/go-hamt-ipld) reference implementation.

## AMT (Array Mapped Trie)

<!-- YAML
added: FIP-0000
changes:
  - fip: FIP-0007
    pr-url: https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0007.md
    description: Upgraded to AMT v3 with performance optimizations including dirty node tracking and optimized ForEach traversals.
-->

The AMT (Array Mapped Trie) is a data structure used in Filecoin for storing sequential data with integer indices. It provides similar benefits to HAMT but optimized for array-like access patterns.

### Key Features
- **Sparse array support**: Efficiently stores arrays with gaps
- **Efficient range queries**: Optimized for sequential access
- **Fixed height**: Predictable performance characteristics

### Version 3 Optimizations
Similar to HAMT v3, the AMT implementation includes:

1. **Dirty node tracking**: Only flushes modified nodes during updates
2. **Optimized traversals**: ForEach operations only load nodes containing keys within the traversal range
3. **Reduced blockstore operations**: Minimizes unnecessary reads and writes

For implementation details, see the [go-amt-ipld](https://github.com/filecoin-project/go-amt-ipld) reference implementation.

## Other Considerations

- The maximum size of an Object should be 1MB (2^20 bytes). Objects larger than this are invalid.
