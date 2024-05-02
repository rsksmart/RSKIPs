---
rskip: 428
title: HSM Segwit Support
created: 23-APR-24
author: NC
purpose: Sca
layer: Core 
complexity: 2
status: Draft
description: 
---

|RSKIP          | 428                |
| :------------ |:-------------------|
|**Title**      | Support HSM Segwit |
|**Created**    | 23-APR-24          |
|**Author**     | NC                 |
|**Purpose**    | Sca                |
|**Layer**      | Core               |
|**Complexity** | 2                  |
|**Status**     | Draft              |

## Abstract

This RSKIP proposes creating a new event that stores the outpoint values of the inputs used in a 
peg-out to enable HSM Segwit support in RSK.

## Motivation

To support HSM Segwit, the PowPeg must provide the outpoint values for signing a segwit peg-out[1].
Currently, selected UTXOs for a peg-out are removed from the available utxos set once 
the peg-out is created. Storing the outpoint values in an event will allow the PowPeg to retrieve 
and provide the outpoint values to the HSM, so being able to sign segwit peg-out properly.

## Specification

The proposal is to create a new event to store the Bitcoin Transaction Hash of the peg-out and 
the outpoint values of each utxo used for it. This event should be emitted whenever a peg-out is 
created, including user peg-outs, migrations, and refunds for rejected transactions.

As described, this new event is composed of two parameters:

First, the Bitcoin Transaction Hash, must be the hash of the peg-out without witnesses and 
signatures.

Second, the list of outpoint values. When dealing with non-segwit peg-outs, the outpoint values will
be an empty list. When dealing with segwit peg-outs, this will be a list of one or more integer 
values in VarInt format[2], ordered in the same order as the inputs of the peg-out transaction.

So, when the PowPeg processes the peg-outs to sign, it will follow a straightforward process. 
It will retrieve the outpoint values from the new event in the rsk transaction where the peg-out 
was created. These values are then provided to the HSM, facilitating the signing of the segwit 
peg-outs.


### Implementation

`segwit_pegout_created` is the name of the new event that will be created. Its signature is:

```
segwit_pegout_created(bytes32 indexed btcTxHash, bytes utxoOutpointValues)
```

As we see, this event consists of two params:

- `btcTxHash` parameter represents the transaction hash without signatures of the pegout or 
migration transaction created. This param is limited to 32 bytes, which is the size of any Bitcoin
transaction hash. This parameter is indexed and allows search queries by a given Bitcoin transaction hash.


- `utxoOutpointValues` parameter represents the outpoint values used for the pegout transaction. 
The order must match the inputs order for the pegout transaction. This parameter stores a list of 
integers in VarInt format serialized as a byte array. The list of outpoint values is serialized and
deserialized using VarInt encoding/decoding mechanism. The order of the outpoint values is preserved
when serializing and deserializing the list of values.

This event should be emitted every time a new peg-out is created.


## Rationale

To make the PowPeg nodes able to retrieve the outpoint values used in a segwit peg-out transaction, 
so they can be provided to the HSM Segwit in order to sign the given peg-out.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] Peg-out efficiency improvement (Segwit) https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md

[2] VarInt https://wiki.bitcoinsv.io/index.php/VarInt

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
