---
rskip: 459
title: Mark rejected peg-ins as processed
created: 10-DEC-24
author: MI
purpose: Usa
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |459           |
| :------------ |:-------------|
|**Title**      |Mark rejected peg-ins as processed |
|**Created**    |10-DEC-24 |
|**Author**     |MI |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

Peg-in transactions rejected by the Bridge should be marked as processed, to ensure that the `rejected_pegin` event is emitted only once per transaction.

## Motivation

A Bitcoin transaction that sends funds to the PowPeg address and is not registered as a transaction created by the Bridge **[1]** is considered a **peg-in** transaction. For this transaction to be considered **valid** by the Bridge contract consensus rules it needs to meet the following criteria.

- Value sent to the PowPeg address above or equal to a minimum of **0.005 BTC**. If there are multiple outputs to the PowPeg address in the same transaction, each output value needs to be above or equal to this amount.
- In case of a peg-in v1 **[2]** the payload in the OP_RETURN output needs to have the format specified in [RSKIP170](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP170.md).
- In a peg-in legacy, the sender address needs to be of type legacy (P2PKH) or segwit-compatible (P2SH-P2WPKH).

If the transaction meets this criteria, then the corresponding amount of RBTC is transferred in the Rootstock network from the Bridge contract to the receiver address. An event called `pegin_btc` is emitted, and the transaction is marked as processed in the Bridge to ensure it is only processed once.

If the criteria are not met, then the peg-in is rejected by the Bridge and an event called `rejected_pegin` is emitted. If possible, a refund is made to the sender and the transaction is marked as processed in the Bridge. If not possible, then an event called `unrefundable_pegin` is emitted but the transaction in this case is not marked as processed **[3]**. This allows the transaction to be re-registered and the events re-emitted multiple times. 


## Specification

When a peg-in is rejected and nonrefundable, mark the transaction as processed after emitting `rejected_pegin` and `unrefundable_pegin` events.


## Rationale

A rejected peg-in transaction should still be considered processed since the corresponding rejection events are emitted as a result of the processing.


## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.


## References

[1] [RSKIP379 - Bridge peg-out and migration transactions index](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP379.md) 

[2] [RSKIP170 - Peg-in to any address](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP170.md) 

[3] [RSKIP181 - Peg-in rejection events](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP181.md) 

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
