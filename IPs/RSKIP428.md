---
rskip: 428
title: New pegout creation event including UTXO outpoint values
created: 23-APR-24
author: NC, MI
purpose: Sca, Sec
layer: Core 
complexity: 1
status: Draft
description: 
---

|RSKIP          | 428                                                      |
| :------------ |:---------------------------------------------------------|
|**Title**      | New pegout creation event including UTXO outpoint values |
|**Created**    | 23-APR-24                                                |
|**Author**     | NC, MI                                                   |
|**Purpose**    | Sca, Sec                                                 |
|**Layer**      | Core                                                     |
|**Complexity** | 1                                                        |
|**Status**     | Draft                                                    |

## Abstract

This RSKIP proposes creating a new event to be emitted when a peg-out transaction is created. This event will include the outpoint values of the inputs used in the peg-out transaction as well as the hash of the Bitcoin transaction without signatures. This information will then be used by the PowHSM to build the sighash and sign segwit pegout transactions.

## Motivation

For the PowHSM to sign segwit peg-outs[1], the outpoint values of the inputs must be provided since they are needed for calculating the sighash of a segwit Bitcoin transaction[2]. Currently, selected UTXOs for a peg-out are removed from the available UTXOs set in the Bridge once the peg-out is created, meaning they cannot be retrieved later. Storing the outpoint values in an event, when peg-outs are created and before the selected UTXOs are removed from the set, is a solution that will allow the PowPeg to retrieve and provide the required outpoint values to the PowHSM to sign segwit peg-outs.

## Specification

The proposal is to create a new event to store the Bitcoin transaction hash without signatures and the UTXO outpoint values of a transaction spending funds from the PowPeg address. This could be either a peg-out, a migration, or a refund transaction. The event will be emitted whenever any of these transactions are created.

This is the proposal signature of the new event which is composed of two parameters:

```
pegout_transaction_created(bytes32 indexed btcTxHash, bytes utxoOutpointValues)
```

The event will be named as `pegout_transaction_created` based on that it will be emitted every time a new peg-out is created.

As we see this event consists of two params:

First, `btcTxHash`, represents the Bitcoin transaction hash, must be the hash of the peg-out without witnesses and without signatures. This param is limited to 32 bytes, which is the size of any Bitcoin transaction hash. This parameter is indexed and allows search queries by a given Bitcoin transaction hash.

Second, `utxoOutpointValues`, represents the list of UTXO outpoint values of the spend transaction. This can be a list of one or more integer values in satoshis stored using VarInt format[3], ordered in the same order as the inputs of the peg-out. This parameter stores a byte array which is the serialized representation of the list of outpoint values in satoshis. This list is serialized and deserialized by using VarInt encoding/decoding mechanism for each entry of the list. The order of the outpoint values is preserved when serializing and deserializing the list.

## Rationale

To make the PowPeg able to retrieve the outpoint values used in a segwit peg-out transaction, so they can be provided to the PowHSM to sign the given segwit peg-out.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] Peg-out efficiency improvement (Segwit) https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md

[2] Transaction Signature Verification for Version 0 Witness Program https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki#specification

[3] VarInt https://wiki.bitcoinsv.io/index.php/VarInt

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
