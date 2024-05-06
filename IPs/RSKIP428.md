---
rskip: 428
title: Emit a new event with the utxo outpoints when a peg-out is created
created: 23-APR-24
author: NC
purpose: Sca
layer: Core 
complexity: 1
status: Draft
description: 
---

|RSKIP          | 428                                                                |
| :------------ |:-------------------------------------------------------------------|
|**Title**      | Emit a new event with the utxo outpoints when a peg-out is created |
|**Created**    | 23-APR-24                                                          |
|**Author**     | NC                                                                 |
|**Purpose**    | Sca                                                                |
|**Layer**      | Core                                                               |
|**Complexity** | 1                                                                  |
|**Status**     | Draft                                                              |

## Abstract

This RSKIP proposes creating a new event that stores the outpoint values of the inputs used in a peg-out to enable the PowHSM sign segwit peg-outs.

## Motivation

For the PowHSM sign segwit peg-outs[1], the outpoint values of the inputs must be provided since they are needed for calculating the sighash of a segwit bitcoin transaction. Currently, selected UTXOs for a peg-out are removed from the available utxos set once the peg-out is created, meaning it is not easy to get them to retrieve the outpoint values for a given segwit peg-out. Storing the outpoint values in an event, when a peg-out is created and before the selected utxos are discarded, is a solution that will allow the PowPeg to retrieve and provide the required outpoint values to the PowHSM to be able to sign a given segwit peg-out.

## Specification

TThe proposal is to create a new event to store the bitcoin transaction hash of the peg-out and the outpoint values of each utxo used as input of the given peg-out. Then, this event will be emitted whenever a transaction spending from the federation address is created. In other words, when a peg-out is created.

This new event is composed of two parameters:

First, the bitcoin transaction hash, must be the hash of the peg-out without witnesses and signatures.

Second, the list of utxo outpoint values. When dealing with non-segwit peg-outs, the outpoint values will be an empty list. When dealing with segwit peg-outs, this will be a list of one or more integer  values in VarInt format[2], ordered in the same order as the inputs of the peg-out.

Therefore, when the PowPeg node processes the peg-outs to sign, it will follow a straightforward process. It will retrieve the outpoint values from the new event in the rsk transaction where the peg-out was created. Then these values will be provided to the PowHSM to sign the given segwit peg-out.

### Implementation

`segwit_pegout_created` is the name of the new event that will be created. Its signature is:

```
segwit_pegout_created(bytes32 indexed btcTxHash, bytes utxoOutpointValues)
```

As we see, this event consists of two params:

- `btcTxHash` parameter represents the bitcoin transaction hash without signatures of the pegout. This param is limited to 32 bytes, which is the size of any Bitcoin transaction hash. This parameter is indexed and allows search queries by a given bitcoin transaction hash.

- `utxoOutpointValues` parameter represents the utxo outpoint values used for the pegout transaction. The order must match the inputs order of the pegout. This parameter stores a list of integers in VarInt format serialized as a byte array. The list of outpoint values is serialized and deserialized using VarInt encoding/decoding mechanism. The order of the outpoint values is preserved when serializing and deserializing the list of values.

Last, This event must be emitted every time a new peg-out is created.

## Rationale

To make the PowPeg able to retrieve the outpoint values used in a segwit peg-out transaction, so they can be provided to the PowHSM to sign the given segwit peg-out.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] Peg-out efficiency improvement (Segwit) https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md

[2] VarInt https://wiki.bitcoinsv.io/index.php/VarInt

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
