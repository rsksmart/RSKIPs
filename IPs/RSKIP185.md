---
rskip: 185
title: Peg-out refund and events
description: 
status: Draft
purpose: Usa
author: JD (@josedahlquist)
layer: Core
complexity: 1
created: 2020-11-19
---
# Peg-out refund and events

|RSKIP          |185           |
| :------------ |:-------------|
|**Title**      |Peg-out refund and events |
|**Created**    |19-NOV-20 |
|**Author**     |JD |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

The Bridge offers a protocol to convert R-BTC back in BTC. This functionality is known as a peg-out.
To do this a user has to send R-BTC to the Bridge and it will start the operation. There is a minimum amount of R-BTCs established that the user has to peg-out at once. In the current protocol, if less than that amount is sent to the Bridge those funds are lost.

This RSKIP proposes improvements in the peg-out protocol that allow refunding users that send less than the minimum required amount. It also proposes the creation of new events to easily known which operations were successful and which ones weren't.

## Motivation

As the platform gains traction, this operation will be more common and users' mistakes become more probable. Therefore we consider it important to improve the protocol to avoid money losses.

## Specification

### Refund

Currently, when a user sends R-BTC to the Bridge it must do it following two directives:
- send it from a EOA account
- send above the minimum (0.008 R-BTC in mainnet)

If the user doesn't follow these rules, the R-BTC transferred will be blocked in the Bridge.

Instead of following this process this RSKIP proposes to refund the user the value sent.

Put simply, when the Bridge receives less than the required amount, it will transfer back the funds to the sender.

### Events

#### release_request_received

```
release_request_received(address indexed sender, bytes btcDestinationAddress, uint amount)
```

- sender: the RSK address of the release requester.
- btcDestinationAddress: the derived BTC address of said RSK address, where the BTC will be transferred to.
- amount: amount in weis the user sent to the Bridge (this doesn't reflect how many BTC will be received since transfer fees will be deducted from that amount).

#### release_request_rejected

```
release_request_rejected(address indexed sender, uint amount, int reason)
```

- sender: the RSK address of the release requester.
- amount: amount in weis the user sent to the Bridge.
- reason: a numeric reason representing why the release was rejected.
- - **1**: amount is less than required.
- - **2**: sender is a contract.

## Rationale

### No refund for contract calls

A contract sending funds to the Bridge is not an acceptable condition and to avoid undetected problems we consider important not refunding the contract. The event will still be generated and the evidence will remain in the blockchain.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
