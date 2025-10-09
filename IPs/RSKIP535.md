---
rskip: 535
title: Add the baseEvent field to the Block header extension
status: Draft
purpose: Sca
author: SDL
layer: Core
complexity: 1
requires: 144, 194,351
created: 2025-10-08
---

## Abstract

This RSKIP adds a new field `baseEvent`  to the Rootstock Block header, according to the rules of [RSKIP 194](./RSKIP194). The new field can be used by a new bridge to store peg-out information and allow proving a chain in zero knowledge at a much lower cost than retrieving it from contract storage. 

## Motivation

[RSKIP 194](./RSKIP194) proposes a modification in the block header to allow extending the block header without interfering with thew PowHSMs.

[RSKIP 144](./RSKIP144) adds a new field to the header called `txExecutionSublistsEdges`. 

[RSKIP 351](./RSKIP351) indicates how this field is appended to the extensioin data.

This RSKIP add one additional field `baseEvent` to the block header extension , and also redefines the version number such that both `txExecutionSublistsEdges` and `baseEvent` fields can be included.

The goal of this new field is to store peg-out event information and allow proving cumulative work of the Rootstock chain in zero knowledge, where the chain refers to a specific peg-out event, at a much lower cost. The alternative requires retrieving the event from contract storage for each confirmation block, which involves many Keccak hash function evaluations. 



## Specification

a new block header version 2 is defined. All blocks post network upgrade must use this version number.


As in version 1, there are two separate serialization methods, one for the transfer of the block header information and the other for the computation of the block hash.

To serialize in version 2, a new field is added to the block header extension. This affects all headers past and future.

A new field is added to the  Block Header Extension:

- For version 2:
  ```
  blockHeaderExtensionEncoded = RLP(logsBloom, txExecutionSublistsEdges,baseEvent
  blockHeaderExtensionHash = Keccak256(RLP(
    Keccak256(logsBloom), txExecutionSublistsEdges,baseEvent))
  ```

If parallel transaction is not activated, then the value `txExecutionSublistsEdges` is left empty.

This RSKIP does not specify how the content of `baseEvent` is selected by the miners, not how the content is verified by the consensus code (these tules will be specified in a separate RSKIP).

However, to prevent large blocks being forwarded, a rule establishes a maximum size of 128 bytes for the `baseEvent` field.

## Rationale

RSKIP 144 and RSKIP351 defines how new fields are added using an header extension and how the block header is serialized. RSKIP351 also adds a new field to help parallel transaction processing. This RSKIP adds another field to the header extension and defines a new version number higher than the previous one.
 
Alternatively, we could use a bitmap for the version field, so that each bit specifies the presence (or absence) of each conditional field in the block header extension. However, this complicates consensus code when parsing the header as allows all field subsets to exist. If no parallel transaction processing is activated, leaving the field `txExecutionSublistsEdges` empty and additing a new element to the RLP list is simpler.


## Backwards compatibility

- This change requires a network upgrade. All full nodes must be upgraded
- RPC responses related to block queries will only add new fields to the returned JSON payload. RPC clients do not need to be updated.
- Powpeg nodes no not need to decode the new field, nor the HSM firmware needs to be upgraded.

## Security considerations

No new security issue was identified related to this RSKIP.

_Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)._
