---
rskip: 327
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


|RSKIP          | 327 |
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

At the moment clients don't have an easy way to know if a pegout has enough confirmations and is already waiting for signatures. With this new event, clients will be able to subscribe to this event and get notified whenever a pegout is confirmed and moved to waiting for signature.

An example of this is the 2wp-app/2wp-api. This tool informs users on the status of their pegout. Right now, to know if a pegout has been confirmed and is waiting for signature this tool makes intricate logic.

## Specification

### Event signature and description

```
pegout_confirmed(address indexed sender, bytes32 indexed originatingRskTx, bytes btcDestinationAddress, uint amount)
```

- `sender`: the RSK address of the requester (the address of the pegnatory that called the `Bridge::updateCollections`).
- `originatingRskTx`: the rsk tx hash (sender) of the previous pegout status.
- `btcDestinationAddress`: the derived BTC address of said RSK address, where the BTC will be transferred to.
- `amount`: amount in weis the user sent to the Bridge (this doesn't reflect how many BTC will be received since transfer fees will be deducted from that amount).

## Rationale

### originatingRskTx field

Sometimes there is no reasonable way to link an event to the original pegout request.
For example, when the pegout is first requested by a user (sending some balance to the Bridge) and is not rejected, then it will be in the `pegoutRequests` structure with data like the following:

```js
// PegoutRequest 
{
    destinationAddressHash160: 'cab5925c59a9a413f8d443000abcc5640bdf0675',
    amountInSatoshis: '531000',
    rskTxHash: 'ea31ee6c7b3de68c34ebcc1c2e0b13f8183060017fe9807aa49cf2035108158c'
}
```

Then, after the next `updateCollections`, it will pass to the `releaseTransactionSet` structure with data like the following:

```js
// PegoutWaitingConfirmation
{
    btcRawTx: '0200000001601...8700000000',
    pegoutCreationBlockNumber: '3048058',
    rskTxHash: 'ea31ee6c7b3de68c34ebcc1c2e0b13f8183060017fe9807aa49cf2035108158c'
}
```

We notice that both of these pegout statuses in the Bridge share the same `rskTxHash`. But then, when it moves to the `rskTxsWaitingForSignatures` structure, we will lose the original `rskTxHash` value and it will now be the tx hash of the transaction that the pegnatory just did that made the bridge move the pegout to the `rskTxsWaitingForSignatures`, losing the connection to the original request. So, to be able to link this new event to the original request, I propose this new field `originatingRskTx` to be added to the event, and emit this event when the Bridge moves the pegout to the `rskTxsWaitingForSignatures` as usual.

## References

[1] Other RSKIP https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP185.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
