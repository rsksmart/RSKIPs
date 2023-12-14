---
rskip: 379
title: Bridge peg-out and migration transactions index
created: 12-DEC-23
author: MI
purpose: Sca,Sec
layer: Core 
complexity: 2
status: Draft
description: 
---

|RSKIP          |379           |
| :------------ |:-------------|
|**Title**      |Bridge peg-out and migration transactions index |
|**Created**    |12-DEC-23 |
|**Author**     |MI |
|**Purpose**    |Sca,Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes a new mechanism to simplify how the Bridge contract identifies incoming transactions as peg-in, peg-out, migration, or non-Bridge related.

## Motivation

The Bridge contract allows any user to register a Bitcoin transaction that sends funds to the PowPeg address by calling `registerBtcTransaction` method. Once this transaction is received by the Bridge, it tries to identify what type of transaction it is.
- **Peg-in**: Transaction that sends funds from an unknown Bitcoin address to the PowPeg. The Bridge registers the UTXO to be used in future peg-out transactions and transfers the corresponding RBTC to the user.
- **Peg-out or migration**: Transaction that sends funds from the currently active or retiring PowPeg address to the active PowPeg address. Could be a migration transaction generated during a PowPeg composition change, or a peg-out transaction that includes a change output to the PowPeg. In this case, the Bridge simply registers the UTXO sent to the PowPeg to be used in future peg-out transactions.
- **Non-Bridge related**: A transaction that does not contain any output to the active or retiring PowPeg address, it is simply ignored by the Bridge.

The current mechanism used by the Bridge to detect the type of transaction is a bit complex and hard to scale. This is caused by the fact that transactions can come from different sources like regular peg-ins, flyover peg-ins, peg-ins to a retiring PowPeg, migration transactions, regular peg-outs, peg-outs created by the retiring federation, etc.

This RSKIP proposes a simplified way for the Bridge to identify the type of a transaction received via `registerBtcTransaction` method that works the same for any given transaction.

## Specification

The proposal is to have an index in the storage of the Bridge contract to save the sighash [1] of the first input of each peg-out or migration transaction created by the Bridge. This provides a simple way for the Bridge to identify transactions that were created by the Bridge contract (peg-out or migration transactions).

When a transaction is received via `registerBtcTransaction` method, the Bridge can then calculate the sighash of the first input of the transaction and check if it exists in the index. If it does, then that means that the transaction was created by the Bridge and is either a peg-out or a migration transaction. UTXOs sending funds to the PowPeg address should be registered in the Bridge, and the transaction should be marked as processed. If the sighash does not exist in the index, but the transaction sends funds to the PowPeg address, then it is a peg-in and should be processed as such. Finally, if the sighash does not exist in the index and the transaction has no outputs to the PowPeg address, then simply ignore the transaction since it's not related to the Bridge.

### Implementation

Every time the Bridge creates a peg-out or migration transaction, it calculates the sighash value of the first input and stores it in a new cell in the Bridge storage. The key of the storage cell should be composed of the prefix `pegoutTxSigHash-` and followed by the corresponding sighash. The value to store is simply a byte value `1`, this value is eventually irrelevant since if the cell exists that means that the transaction was created by the Bridge.

## Rationale

The reason to identify transactions by the sighash of the first input instead of using the transaction id is because of transaction malleability [2]. The Bridge uses the flag SIGHASH_ALL when signing transactions, so each input’s signature is only valid if all other inputs and outputs remain unchanged.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] SIGHASH (https://wiki.bitcoinsv.io/index.php/SIGHASH_flags)

[2] Transaction malleability (https://en.bitcoin.it/wiki/Transaction_malleability)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
