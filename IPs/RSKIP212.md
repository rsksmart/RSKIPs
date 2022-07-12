---
rskip: 212
title: HW-compatible Transaction Versioning System
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2021-01
---
# HW-compatible Transaction Versioning System


|RSKIP          | 212 |
| :------------ |:-------------|
|**Title**      |HW-compatible Transaction Versioning System|
|**Created**    |JAN-2021 |
|**Author**     | SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |


# **Abstract**

This RSKIP proposes the addition of a version field to transactions maintaining high compatibility with existent hardware wallets, and requiring minimum changes from software wallets.

## Motivation

Many features, like ephemeral calldata, multi-signed transactions and LTCP transaction compression require adding more information to transactions. However currently RSK transactions do not have a way to indicate a new transaction format is being used. We propose a transaction format that embeds a version field in the current nonce field, keeping maximum compatibility with wallets, specially hardware wallets.



## Specification

Currently the transaction contains the following fields, stored using RLP:

1. nonce
2. gasPrice
3. gasLimit
4. ReceiveAddress
5. value
6. data
7. v
8. r
9. s

When serialized, each field begins with an RLP identifier that indicates if the item is an element or a list of elements. This format will be called version 0.

For version 1, the constant (1<<32) is added to the nonce, and the field is renamed nonceAndVersion, to avoid confusion. As most transactions have nonces below 256, the change implies adding 3 zero bytes of padding to the serialized format, which results in approximately a 3% overhead. 

The encoded nonceAndVersion must be masked with (1<<32)-1 for the extraction of the nonce value.

The only version value accepted is 1. Following RSKIPs can define additional versions. 

# Rationale

This change was selected because it doesn't break compatibility of most hardware wallets. Another (cleaner) method would be adding a version field to the transaction RLP list, which doesn't bring any overhead, but requires parsing the whole RLP encoded transaction before the version can be detected.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 

## Test Cases

TBD

## Implementation

TBD


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


