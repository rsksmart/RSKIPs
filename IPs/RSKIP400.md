---
rskip: 400
title: Calldata gas cost reduction
description: 
status: Draft
purpose: Sca
author: VK
layer: Core
complexity: 2
created: 2023/07/12
---
# Calldata gas cost reduction


|RSKIP          | 400 |
| :------------ |:-------------|
|**Title**      |Calldata gas cost reduction|
|**Created**    |JUL-2023 |
|**Author**     |VK |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |


## Abstract

This improvement implements Ethereum [EIP-2028: Transaction data gas cost reduction](https://eips.ethereum.org/EIPS/eip-2028)

## Motivation

By lowering gas cost of calldata, this change improves scalability in general, as more data can fit within a single
block. Additionally, it's especially useful for layer two scaling solutions like rollups.

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [EIP-2028](https://eips.ethereum.org/EIPS/eip-2028)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
