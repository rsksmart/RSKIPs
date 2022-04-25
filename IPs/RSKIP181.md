---
rskip: 181
title: Peg-in rejection events
description: 
status: Draft
purpose: Usa
author: JD (@josedahlquist)
layer: Core
complexity: 2
created: 2020-11-03
---

# Peg-in rejection events

|RSKIP          |181           |
| :------------ |:-------------|
|**Title**      |Peg-in rejection events |
|**Created**    |03-NOV-20 |
|**Author**     |JD |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes changes to the Bridge peg-in protocol in order to register evidence on-chain when a peg-in was rejected and why it happened.

## Motivation

These changes will improve the user experience of the peg-in.

## Specification

When a peg-in can't be properly processed (i.e. R-BTC get transferred to a destination address), the registerBtcTransaction finishes with no information on the reason of this happening. Post-mortem analysis may help determine the reason, but this is a bad UX in general.

This RSKIP proposes adding new events that help the consumers understand why the peg-in "failed".

### rejected_pegin event

```
rejected_pegin(bytes32 indexed btcTxHash, int reason)
```

This event will be generated any time a peg-in won't transfer R-BTC to the destination address.

#### Data

- btcTxHash: matches the peg-in BTC transaction hash
- reason: a numeric value indicating the reason for the rejection. see [rejection reason](#reason)

##### Reason

1 → peg-in cap surpassed.
[see the RSKIP-134](#References)

2 → legacy peg-in multisig sender.
If the sender is a BTC multisig address, the recipient can't be determined, therefore the R-BTC won't be transferred.

3 → legacy peg-in undetermined sender.
If the sender is not a valid BASE58 address (i.e. BECH32), the sender can't be determined.

4 → pegin v1 invalid payload.
If the payload doesn't respect the pegin v1 protocol ([see the RSKIP-170](#References)), the recipient can't be determined.

### unrefundable peg-in event

```
unrefundable_pegin(bytes32 indexed btcTxHash, int reason)
```

#### Data

- btcTxHash: matches the peg-in BTC transaction hash
- reason: a numeric value indicating the reason for peg-in to not be refundable. see [unrefundable reason](#reason)

##### Reason

1 → legacy peg-in unidentified sencer.
If the sender is not a valid BASE58 address (i.e. BECH32), the sender can't be determined, therefore the BTC can't be refunded.

2 → pegin v1 refund address not set, unidentified sender.
If the payload doesn't include a refund address ([see the RSKIP-170](#References)), and the sender is not a valid BASE58 address, the BTC can't be refunded.

## References

[1] [RSKIP-134 - lockingcap](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP134.md)

[2] [RSKIP-170](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP170.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
