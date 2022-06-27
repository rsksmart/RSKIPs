---
rskip: 213
title: Simple Transaction Versioning System
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2021-02
---
# Simple Transaction Versioning System


|RSKIP          | 213 |
| :------------ |:-------------|
|**Title**      |Simple Transaction Versioning System|
|**Created**    |FEB-2021 |
|**Author**     |SDL|
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1|
|**Status**     |Draft |


# **Abstract**

This RSKIP proposes the addition of a version field so that transaction parsing can start after the transaction version has been verified. This reduces the amount of changes required for the node and software wallets.

## Motivation

Many features, like ephemeral calldata, multi-signed transactions and LTCP transaction compression require adding more information to transactions. However currently RSK transactions do not have a way to indicate that a new transaction format is being used. We propose a transaction format that embeds a version field in the current nonce position, keeping high compatibility with wallets.



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


### Transaction Format  Version 1

The transaction format version 1 adds the new fields version embedded in the position previously used by the nonce. 

1. multiPurpose = (version, nonce)
2. gasPrice
3. gasLimit
4. receiveAddress
5. value
6. data
7. v
8. r
9. s

The field nonce is renamed multiPurpose, and it contains an RLP list of two values: version and nonce. Version is encoded as an integer. The only version value accepted is 1. The encoding must be minimal (no zero padding). Following RSKIPs can define additional versions. Transactions version 0 must use the previous format.  The RLP parser can detect if it is a transaction version 0 or version 1 by inspecting only the multiPurpose field (previously called nonce). If this element is a list (more precisely, starts with byte 0xc2), then the version is 1. If not, then it is the nonce, and the transaction version is 0.

When a transaction with version 1 is received, and it needs retransmission, it must be retransmitted in transaction format 1. A transaction version 0 is retransmitted a transaction format 0.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 

## Test Cases

TBD

## Implementation

TBD


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


