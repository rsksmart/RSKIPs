---
rskip: 535
title: Add the baseEvent field to the Block header extension
status: Adopted (Testnet)
purpose: Sca
author: SDL
layer: Core
complexity: 1
requires: 144, 194,351
created: 2025-10-08
---

# Add the baseEvent field to the Block header extension

## Abstract

This RSKIP adds a new field `baseEvent`  to the Rootstock Block header, according to the rules of [RSKIP 194](./RSKIP194.md). The new field can be used by a new bridge to store peg-out information and allow proving a chain in zero knowledge at a much lower cost than retrieving it from contract storage. 

## Motivation

[RSKIP 194](./RSKIP194.md) proposes a modification in the block header to allow extending the block header without interfering with the PowHSMs.

[RSKIP 144](./RSKIP144.md) adds a new field to the header called `txExecutionSublistsEdges`. 

[RSKIP 351](./RSKIP351.md) indicates how this field is appended to the extension data.

This RSKIP add one additional field `baseEvent` to the block header extension , and also redefines the version number to 2 such that both `txExecutionSublistsEdges` and `baseEvent` fields can be included.

The goal of this new field is to store peg-out event information and allow proving cumulative work of the Rootstock chain in zero knowledge, where the chain refers to a specific peg-out event, at a much lower cost. The alternative requires retrieving the event from contract storage for each confirmation block, which involves many Keccak hash function evaluations. 



## Specification

a new block header version 2 is defined. All blocks post network upgrade must use this version number.


As in version 1, there are two separate serialization methods, one for the transfer of the block header information and the other for the computation of the block hash.

To serialize in version 2, a new field is added to the block header extension. This affects all headers past and future.

A new field is added to the  Block Header Extension:

- For version 2:
  ```
  blockHeaderExtensionEncoded = RLP(logsBloom, baseEvent, txExecutionSublistsEdges)
  blockHeaderExtensionHash = Keccak256(RLP(
    Keccak256(logsBloom), baseEvent, txExecutionSublistsEdges))
  ```

The `baseEvent` element is always present. If the block has no base event, `baseEvent` is an empty byte array.

If parallel transaction execution ([RSKIP 144](./RSKIP144.md)) is not active, the `txExecutionSublistsEdges` element is omitted from the list, so the extension has two elements. The two-element form is what nodes produce on a network that activates this RSKIP without RSKIP 144. Today RSKIP 535 is active only on Testnet, where RSKIP 144 is also active, so the extension has three elements there.

### Extended header

When serializing the block header for transmission or storage, version 2 updates the layout defined in [RSKIP 351](./RSKIP351.md). No field moves; one field is added:

* The bloom filter data (256 bytes) stays in its original slot, in place of the extension data.
* The blockVersion field (integer, value 2) (1 byte) follows `ummRoot`, as in version 1.
* `txExecutionSublistsEdges` follows the version field (present only when RSKIP 144 is active), as in version 1.
* `baseEvent` (at most 128 bytes; an empty byte array when the block has no base event) is new. It follows `txExecutionSublistsEdges` and precedes the merged mining fields.

Note that the extended header places `txExecutionSublistsEdges` before `baseEvent`, while the block header extension places `baseEvent` before `txExecutionSublistsEdges`. Only the extension order affects the block hash.

This RSKIP does not specify how the content of `baseEvent` is selected by the miners, nor how the content is verified by the consensus code (these rules will be specified in a separate RSKIP).

However, to prevent large blocks being forwarded, a rule establishes a maximum size of 128 bytes for the `baseEvent` field.

## Rationale

RSKIP 144 and RSKIP351 defines how new fields are added using an header extension and how the block header is serialized. RSKIP351 also adds a new field to help parallel transaction processing. This RSKIP adds another field to the header extension and defines a new version number higher than the previous one.
 
Alternatively, we could use a bitmap for the version field, so that each bit specifies the presence (or absence) of each conditional field in the block header extension. However, this complicates consensus code when parsing the header as allows all field subsets to exist. With a version number, `txExecutionSublistsEdges` is the only element that can be omitted, so the number of elements in the list (2 or 3) tells whether it is present.


## Backwards compatibility

- This change requires a network upgrade. All full nodes must be upgraded
- RPC responses related to block queries will only add new fields to the returned JSON payload. RPC clients do not need to be updated.
- Powpeg nodes do not need to decode the new field, nor the HSM firmware needs to be upgraded.
- In the block header extension, `baseEvent` comes before `txExecutionSublistsEdges`. This is the order active on Testnet since block 7,604,200. Changing it changes the block hash, so it requires a new network upgrade.

## Security considerations

No new security issue was identified related to this RSKIP.

_Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)._
