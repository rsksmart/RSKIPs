---
rskip: 417
title: Avoid transactions to be reverted when Bridge method calls from smart contracts return an empty response
created: 28-FEB-24
author: MI
purpose: Usa
layer: Core 
complexity: 1
status: Draft
description: 
---

|RSKIP          |417           |
| :------------ |:-------------|
|**Title**      |Avoid transactions to be reverted when Bridge method calls from smart contracts return an empty response |
|**Created**    |28-FEB-24 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

When a Bridge method with no return value is called from a smart contract it causes the node to throw a null pointer exception and the transaction to be reverted. This RSKIP proposes a change so that no exception is thrown and the transaction is not reverted.

## Motivation

There are currently 6 Bridge methods that have no return value.

- addSignature
- receiveHeaders
- registerBtcCoinbaseTransaction
- registerBtcTransaction
- releaseBtc
- updateCollections

If any of these methods were to be called from a smart contract, the transaction would get automatically reverted. 

## Specification

When the Bridge executes a method that has no return value, return an empty byte array instead of `null`.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
