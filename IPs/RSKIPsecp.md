---
rskip:
title: Precompiled contracts for +/* on Secp256k1
description: This RSKIP introduces precompiled contracts for elliptic curve operations that are required in order to efficiently perform MuSig2 signature verifications on the Secp256k1 curve.
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner) 
layer: Core
complexity: 2
status: 
created: 2024-04-10
---
# Precompiled contracts for +/* on Secp256k1

|RSKIP          |              |
| :------------ |:-------------|
|**Title**      |Precompiled contracts for +/* on Secp256k1|
|**Created**    |10-APR-2024 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     ||

# Abstract

This RSKIP adds precompiled contracts for addition and scalar multiplication on a specific to the Secp256k1 curve. This will reduce the gas cost and improve robustness of MuSig2 and other signature aggregation algorithms.

# Motivation

The Solidity language has a bigint artichmetic opcodes and a modular exponentiation precompile. However, uting these low level functions to build complex protocols over secp256k1 is inefficient, costly and error-prone. Cryptographic libraries should be re-used and not re-implemented because they are  security-critical. Therefore exposing well-tested cryptographic functions to Solidity is desirable.
For this reason the [EIP-196](https://eips.ethereum.org/EIPS/eip-196) was created. In this RSKIP we propose to introduce precompiles for point addition (ADD)  and scalar multiplication (MUL) on the elliptic curve "secp256k1". Having efficient operations on this curve is essential to enable musig2 functionality in contracts that need to coordinate the aggregatation of signatures for Bitcoin transactions.  




# Specification

Add precompiled contracts for point addition (ADD)  and scalar multiplication (MUL) on the elliptic curve "Secp256k1".

Address of ADD: 0x01000016
Address for MUL: 0x01000017


## Encoding

Fields are encoded in the same way as [EIP-196](https://eips.ethereum.org/EIPS/eip-196). Field elements and scalars are encoded as 32 byte big-endian numbers. Curve points are encoded as two field elements `(x, y)`, where the point at infinity is encoded as `(0, 0)`.

Tuples of objects are encoded as their concatenation.

For both precompiled contracts, if the input is shorter than expected, it is assumed to be virtually padded with zeros at the end (i.e. compatible with the semantics of the `CALLDATALOAD` opcode). If the input is longer than expected, surplus bytes at the end are ignored.

The length of the returned data is always as specified (i.e. it is not "unpadded").

## Exact semantics

Invalid input: For both contracts, if any input point does not lie on the curve or any of the field elements (point coordinates) is equal or larger than the field modulus p, the contract fails. The scalar can be any number between `0` and `2**256-1`.

#### ADD
Input: two curve points `(x, y)`.
Output: curve point `x + y`, where `+` is point addition on the elliptic curve `Secp256k1` .
Fails on invalid input and consumes all gas provided.

#### MUL
Input: curve point and scalar `(x, s)`.
Output: curve point `s * x`, where `*` is the scalar multiplication on the elliptic curve `Secp256k1`.
Fails on invalid input and consumes all gas.

### Gas costs

ECADD gas costs is set according to [EIP-1108](https://eips.ethereum.org/EIPS/eip-1108).
ECMUL gas cost is set to be equal to `ECRECOVER`, as `ECRECOVER` already performs a scalar multiplication in the same curve.

 - Gas cost for ``ECADD``: 150
 - Gas cost for ``ECMUL``: 3000

# Rationale

 The Rootstock client already links to a the libsecp256k1 library to provide the ECRECOVER operation, so extending the node to provide ECADD and ECMUL on this currve does not require linking with additional external libraries. Also the these operations are provided by the BouncyCastle library, which can be used as a reference implementation.

The specific curve `secp256k1` was chosen because it is the curve used by Bitcoin for Schnorr and ECDSA signatures. 

The representation of elliptic curve points was chosen to be compatible with [EIP-196](https://eips.ethereum.org/EIPS/eip-196).


# Backwards Compatibility

As with the introduction of any precompiled contract, this is can only be accomplished with a hard-fork. The addresses were chosen to lie within the range reserved to Rootstock-only precompiles.

## Test Cases

Inputs to test:

 - Curve points which would be valid if the numbers were taken mod p (should fail).
 - Both contracts should succeed on empty input.
 - Truncated input that results in a valid curve point.
 - Points not on curve (but valid otherwise).
 - Multiply point with scalar that lies between the order of the group and the field (should succeed).
 - Multiply point with scalar that is larger than the field order (should succeed).

# Implementation

See https://github.com/rsksmart/rskj/tree/secp_ec for canonincal implementation and testing code. This implementation uses the BouncyCastle cryptographic library, which is slower than libsecp256k1. The gas costs are set accondingly to libsecp256k1 performance.

# Security Considerations

This proposal follows the input/output format of [EIP-196](https://eips.ethereum.org/EIPS/eip-196) to avoid introducing new vulnerabilities related to blank or long inputs and outputs. Solidiy wrappers written for the bn128 precompiles should be easily adapted to work with the new precompiles. 

Gas cost were chosen as [EIP-1108](https://eips.ethereum.org/EIPS/eip-1108) to match real costs, and should not create new DoS vectors.

The usual code for addition and multiplication over secp256k1 is deterministic and does not present resource-consuming edge cases.


The precompiles introduced do not increase the protocol's attack surface as the addition and multiplication functions are already linked by the Rootstock codebase and in use when checking ECDSA signatures. 
 
# Copyright


Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
