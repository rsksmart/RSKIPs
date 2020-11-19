# Title

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
To do this a user has to send R-BTC to the Bridge and it will start the operation.

This RSKIP proposes improvements in this protocol to refund users that send less than the minimum required amount and at the same time proposes to create events to easily known which operations were successful and which ones weren't.

## Motivation

As the platform gains traction this operation will be more common and users mistakes become more probable, therefore we consider important improve the protocol to avoid money loss.

## Specification

### Refund

When a user sends R-BTC to the Bridge it must do it following two directives:
- send it from a EOA account
- send above the minimum (0.008 R-BTC in mainnet)

If the user doesn't follow these rules, the R-BTC transferred will be blocked in the Bridge.

Instead of following this process this RSKIP proposes to refund the user the value sent.

Put simple, when the Bridge receives less than the required amount, it will transfer back the funds.

### Events

#### release_request_received

```
release_request_received(address indexed sender, bytes btcDestinationAddress, uint amount)
```

- sender: the rskAddress of the release requester.
- btcDestinationAddress: the derived BTC address of said rsk address.
- amount: amount in weis the user sent to the Bridge (this doesn't reflect how many BTC will be received).

#### release_request_rejected

```
release_request_rejected(address indexed sender, uint amount, int reason)
```

- sender: the rskAddress of the release requester.
- amount: amount in weis the user sent to the Bridge.
- reason: a numeric reason representing why the release was rejected.
- - **1**: amount is less than required.
- - **2**: caller is a contract.

## Rationale

### No refund for contract calls

A contract sending funds to the Bridge is not an acceptable condition and to avoid undetected problems we consider important not refunding the contract. The event will still be generated and the evidence will remain in the blockchain.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
