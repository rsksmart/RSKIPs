---
rskip: 398
title: PUSH0 instruction
description: 
status: Adopted
purpose: Sca
author: VK
layer: Core
complexity: 2
created: 2023/07/11
---
# PUSH0 instruction


|RSKIP          | 398 |
| :------------ |:-------------|
|**Title**      |PUSH0 instruction|
|**Created**    |JUL-2023 |
|**Author**     |VK |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |


## Abstract

This improvement implements Ethereum [PUSH0 instruction](https://eips.ethereum.org/EIPS/eip-3855)

## Motivation

This change improves scalability (as this opcode requires less gas comparing to other PUSH* opcodes) and compatibility with EVM opcodes.

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [EIP-3855](https://eips.ethereum.org/EIPS/eip-3855)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
