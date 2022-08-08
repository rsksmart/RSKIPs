---
rskip: 346
title: New pegout_confirmed event and add_signature event logging order
description: Add a new event called "pegout_confirmed" that will be emitted when a pegout has reached the required confirmations. Also, check that a new signature was actually added before emitting the add_signature event.
status: Draft
purpose: Usa
author: JT (@jeremy.then)
layer: Core
complexity: 2
created: 2022-07
---
# New pegout_confirmed event


|RSKIP          | 346 |
| :------------ |:-------------|
|**Title**      |New pegout_confirmed event|
|**Created**    |Jul-2022 |
|**Author**     | JT |
|**Purpose**    | Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

##  Abstract

### New pegout_confirmed event

Currently no event is being emitted when a pegout that is waiting for confirmation (in the `releaseTransactionSet` structure) has the required confirmations and is moved to waiting for signatures (to the `rskTxsWaitingForSignatures`).
Right now, to check if a pegout has enough confirmation and that is moved to the `rskTxsWaitingForSignatures` structure, we need to get the state of the bridge and look for the pegout in `releaseTransactionSet` or `rskTxsWaitingForSignatures`, which is not that performant or convenient.

We need to add a new event to the Bridge that will be emitted when the Bridge moves a pegout to the `rskTxsWaitingForSignatures`.
I propose this event is called `pegout_confirmed`.

### add_signature event logging order

At the moment the add_signature event is being emitted before actually adding the signature.

## Motivation

### New pegout_confirmed event

At the moment clients don't have an easy way to know if a pegout has enough confirmations and is already waiting for signatures. With this new event, clients will be able know, probably live, when a pegout is already confirmed and waiting for signatures.

An example of this is the 2wp-app/2wp-api. This tool informs users on the status of their pegout. Right now, to know if a pegout has been confirmed and is waiting for signature this tool makes intricate logic.

### add_signature event logging order

In some cases a signature will not be added to a transaction, so there would not be a need to actually emit the add_signature event.
## Specification

### Event signature and description

This event will exist after the batch pegout consensus changes (HOP, RSKIP271).

```
pegout_confirmed(bytes32 indexed btcTxHash, uint pegoutCreationBlockNumber)
```

- `btcTxHash`: the hash of the Bitcoin transaction without signatures that was just confirmed
- `pegoutCreationBlockNumber`: the block where the pegout transaction was created, this is just informational value and totally optional

### add_signature event logging order

At the end of the `BridgeSupport::addSignature` method, we can see that is calling `eventLogger.logAddSignature` before `BridgeSupport::processSigning`.
So, I propose to remove the `eventLogger.logAddSignature` call from `BridgeSupport::addSignature` and use it inside `BridgeSupport::processSigning` when the signature is actually added to the btcTx.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 

## References

[1] Other RSKIP https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP185.md
[2] Other RSKIP https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
