---
rskip: 351
title: Miniheader - block header compression
status: Draft
purpose: Sca
author: IO (ilan@iovlabs.org)
layer: Core
complexity: 1
requires: 144, 194
created: 2022-09-12
---

## Abstract

This RSKIP proposes a technique to compress the block header. By moving information such as the logs bloom to a new data structure, the block header can be modified and versioned including new information without changing the block header format.

## Motivation

[RSKIP 194](./RSKIP194) proposes a modification in the block header to replace the logs bloom for its hash. This solves the problem of the logs bloom filter in the header allowing nodes to download a lighter version of the header. It also proposes a version number for the block header that simplifies future changes in the data structure, keeping the header without modifications.

[RSKIP 144](./RSKIP144) adds a new field to the header called `txExecutionSublistsEdges`. Including this field in the header extension does not require any firmware upgrade for the federation HSMs.

## Specification

The block header compression is specified similar to RSKIP 194.

### Block header version

All blocks prior the hard-fork block number are considered version 0. Afterward, new blocks must be version 1. Consensus must validate that the blockVersion is exactly 1 for blocks after the hard fork, until this number is incremented by another consensus change.

For block headers version 1, there are two separate serialization methods, one for the transfer of the block header information and the other for the computation of the block hash.

### Compressed header

When serializing the block header to obtain the block hash, the `logBloom` field is renamed `extendedData`. This affects all headers past and future.

- For version 0:
  ```
  extendedData = logBlooms
  ```
- For version 1:
  ```
  extendedData = RLP(version, blockHeaderExtensionHash)
  ```

#### Block Header Extension

A new data structure called Block Header Extension is used to transfer the information that is not included in the compressed header.

- For version 0 is not existent.
- For version 1:
  ```
  blockHeaderExtensionEncoded = RLP(logsBloom, txExecutionSublistsEdges)
  blockHeaderExtensionHash = Keccak256(RLP(
    Keccak256(logsBloom), txExecutionSublistsEdges))
  ```

### Extended header

When serializing the block header for transmission or storage, the data structure includes

* The blockVersion field (integer) (1 byte)
* The bloom filter data (256 bytes).
* `txExecutionSublistsEdges` (at most 8 bytes now, 2 bytes for each short).

## Implementation

The logs bloom field allows new space in the block header for adding the hash of the extra data plus the version number. By using this space, the federators' HSMs do not need firmware upgrades.

## Rationale

RSKIP 144 introduces a new field on the block header. It is a good opportunity to build this data structure correctly.

This implementation is a first intention of refurbishing the header to allow smoother modifications in the future. A better implementation would completely rebuild the block header "cleaning up" the unnecessary used space, moving fields like the minimum gas price to the new data structure, as described in RSKIP 194.

Block header extension hash uses `Keccak256(logsBloom)` because it allows to verify the structure without requesting logs bloom that is 256 bytes long. If necessary, a method `getHeaderExtensionCompressed` can be created. For future versions, we recommend to hash any field that has length >= 32 bytes (keccak256 output size).

## Backwards compatibility

- Requires hard fork. All full nodes must be updated
- RPC responses will only add new fields to the payload. RPC clients do not need to be updated.
- The RLP encoding of the fields used by the federation is not modified. HSM firmware does not need any upgrade

## Security considerations

No new security issue was identified related to this RSKIP.

_Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)._
