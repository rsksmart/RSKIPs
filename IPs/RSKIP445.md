---
rskip: 445
title: MCOPY instruction
description: 
status: Draft
purpose: Usa
author: AE
layer: Core
complexity: 2
created: 2024/08/12
---
# MCOPY instruction


|RSKIP          | 445 |
| :------------ |:-------------|
|**Title**      |MCOPY instruction|
|**Created**    |AUG-2024 |
|**Author**     |AE |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |


## Abstract

This improvement implements Ethereum [MCOPY instruction](https://eips.ethereum.org/EIPS/eip-5656)

## Motivation

This change introduces the MCOPY instruction, an efficient instruction for copying memory areas, to Rootstock's VM. This improves compatibility with EVM opcodes.

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [EIP-5656](https://eips.ethereum.org/EIPS/eip-5656)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
