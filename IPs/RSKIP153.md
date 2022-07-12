---
rskip: 153
title: Add BLAKE2 Compression Function `F` Precompile 
description: 
status: Accepted
purpose: Usa
author: FJ (@fedejinich)
layer: Core
complexity: 2
created: 2020-11-19
---
# Add BLAKE2 Compression Function `F` Precompile

|RSKIP          |153           |
| :------------ |:-------------|
|**Title**      |Add BLAKE2 Compression Function `F` Precompile |
|**Created**    |19-11-2020 |
|**Author**     |FJ |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Accepted |

## Abstract

This RSKIP introduces a new precompiled contract which implements the compression function F used in the BLAKE2 cryptographic hashing algorithm, for the purpose of allowing interoperability between the EVM and Zcash, as well as introducing more flexible cryptographic hash primitives to the EVM.

Also this RSKIP will make RSK compatible with [EIP-152](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-152.md).

## Motivation

Besides being a useful cryptographic hash function and SHA3 finalist, BLAKE2 allows for efficient verification of the Equihash PoW used in Zcash, making a BTC Relay - style SPV client possible on RSK. A single verification of an Equihash PoW verification requires 512 iterations of the hash function, making verification of Zcash block headers prohibitively expensive if a Solidity implementation of BLAKE2 is used.

BLAKE2b, the common 64-bit BLAKE2 variant, is highly optimized and faster than MD5 on modern processors.

Interoperability with Zcash could enable contracts like trustless atomic swaps between the chains, which could provide a much needed aspect of privacy to the very public RSK blockchain.

## Specification

This new precompiled contract should be enabled when `blockNumber >= irisHardfork`.

The rest of the specification is described in [EIP-152](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-152.md).

## References

[1] [EIP-152](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-152.md)

[2] [BLAKE2 F](https://tools.ietf.org/html/rfc7693#section-3.2)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
