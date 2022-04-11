---
rskip: 188
title: Precompiled Contracts for BLS12-381 Curve Operations 
description: 
status: Draft
purpose: Usa
author: FJ (@fedejinich)
layer: Core
complexity: 2
created: 2020-11-20
---

# Precompiled Contracts for BLS12-381 Curve Operations

|RSKIP          | 188 |
| :------------ |:-------------|
|**Title**      |Precompiled Contracts for BLS12-381 Curve Operations |
|**Created**    |20-11-2020 |
|**Author**     |FJ |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

In order to mantain Ethereum compatibility, we introducue a set of precompiled contracts. Enabling efficient operations on BLS12-381 curve, in a necessary set for BLS signature verifications and SNARKs verifications.

### Proposed addresses table

| Precompile | Address |
| :-------------| -----:|
| BLS12_G1ADD | 0x0a | 
| BLS12_G1MUL | 0x0b |
| BLS12_G1MULTIEXP | 0x0c |
| BLS12_G2ADD | 0x0d |
| BLS12_G2MUL | 0x0e |
| BLS12_G2MULTIEXP | 0x0f |
| BLS12_PAIRING | 0x10 |
| BLS12_MAP_FP_TO_G1	| 0x11 |
| BLS12_MAP_FP2_TO_G2 | 0x12 |

## Motivation

Motivation of this precompile:
- Mantain compatibility between RSK and Ethereum
- Add a cryptographic primitive that allows to get 120+ bits of security for operations over pairing friendly curve compared to the existing BN254 precompile that only provides 80 bits of security.

## Specification

If `blockNumber >= N` (to be defined) we introduce nine separate precompiles to perform the following operations:

`BLS12_G1ADD` - to perform point addition on a curve defined over prime field
`BLS12_G1MUL` - to perform point multiplication on a curve defined over prime field
`BLS12_G1MULTIEXP` - to perform multiexponentiation on a curve defined over prime field
`BLS12_G2ADD` - to perform point addition on a curve twist defined over quadratic extension of the base field
`BLS12_G2MUL` - to perform point multiplication on a curve twist defined over quadratic extension of the base field
`BLS12_G2MULTIEXP` - to perform multiexponentiation on a curve twist defined over quadratic extension of the base field
`BLS12_PAIRING` - to perform a pairing operations between a set of pairs of (G1, G2) points
`BLS12_MAP_FP_TO_G1` - maps base field element into the G1 point
`BLS12_MAP_FP2_TO_G2` - maps extension field element into the G2 point

Mapping functions are implemented according to IEFT specification version `v7`(!) using an simplified SWU method. It does NOT perform mapping of the byte string into field element that can be implemented in many different ways and can be efficiently performed in EVM, but only does field arithmetic to map field element into curve point. Such functionality is required for signature schemes.

Multiexponentiation operation is included to efficiently aggregate public keys or individual signer’s signatures during BLS signature verification.

The rest of the specification it's described at [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537)

## Rationale

Motivation section covers a total motivation to have operations over BLS12-381 curve available. We also extend a rationale for move specific fine points.

Multiexponentiation as a separate call

Explicit separate multiexponentiation operation that allows one to save execution time (so gas) by both the algorithm used (namely Peppinger algorithm) and (usually forgotten) by the fact that `CALL` operation in Ethereum is expensive (at the time of writing), so one would have to pay non-negigible overhead if e.g. for multiexponentiation of `100` points would have to call the multipication precompile `100` times and addition for `99` times (roughly `138600` would be saved).

## Backwards Compatibility

There are no backward compatibility questions.

## Test Cases

Due to the large test parameters space we first provide properties that various operations must satisfy. We use additive notation for point operations, capital letters (P, Q) for points, small letters (a, b) for scalars. Generator for G1 is labeled as G, generator for G2 is labeled as H, otherwise we assume random point on a curve in a correct subgroup. 0 means either scalar zero or point of infinity. 1 means either scalar one or multiplicative identity. group_order is a main subgroup order. e(P, Q) means pairing operation where P is in G1, Q is in G2.

### Requeired properties for basic ops (add/multiply):

- Commutativity: `P + Q = Q + P`
- Additive negation: `P + (-P) = 0`
- Doubling: `P + P = 2*P`
- Subgroup check: `group_order * P = 0`
- Trivial multiplication check: `1 * P = P`
- Multiplication by zero: `0 * P = 0`
- Multiplication by the unnormalized scalar: `(scalar + group_order) * P = scalar * P`

### Required properties for pairing operation:
- Degeneracy: `e(P, 0*Q) = e(0*P, Q) = `1
- Bilinearity: `e(a*P, b*Q) = e(a*b*P, Q) = e(P, a*b*Q)` (internal test, not visible through ABI)

Test vector for all operations are expanded in this csv files in repo.

## References

[1] [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
