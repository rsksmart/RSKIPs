---
rskip: 346
title: New pegout_confirmed event
description: Add a new event called "pegout_confirmed" that will be emitted when a pegout has reached the required confirmations
status: Draft
purpose: Sec, Usa
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
|**Purpose**    |Sec, Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

#  **Abstract**

Currently no event is being emitted when a pegout that is waiting for confirmation (in the `releaseTransactionSet` structure) has the required confirmations and is moved to waiting for signatures (to the `rskTxsWaitingForSignatures`).
Right now, to check if a pegout has enough confirmation and that is moved to the `rskTxsWaitingForSignatures` structure, we need to get the state of the bridge and look for the pegout in `releaseTransactionSet` or `rskTxsWaitingForSignatures`, which is not that performant or convenient.

We need to add a new event to the Bridge that will be emitted when the Bridge moves a pegout to the `rskTxsWaitingForSignatures`.
I propose this event is called `pegout_confirmed`.

## Motivation

At the moment clients don't have an easy way to know if a pegout has enough confirmations and is already waiting for signatures. With this new event, clients will be able know, probably live, when a pegout is already confirmed and waiting for signatures.

An example of this is the 2wp-app/2wp-api. This tool informs users on the status of their pegout. Right now, to know if a pegout has been confirmed and is waiting for signature this tool makes intricate logic.

## Specification

### Event signature and description

```
pegout_confirmed(bytes32 indexed btcTxHash, uint pegoutCreationBlockNumber)
```

- `btcTxHash`: the hash of the Bitcoin transaction without signatures that was just confirmed
- `pegoutCreationBlockNumber`: the block where the pegout transaction was created, this is just informational value and totally optional

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 

## References

[1] Other RSKIP https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP185.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
