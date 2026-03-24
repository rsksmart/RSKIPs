---
rskip: 551
title: Deprecate RSKIP459
created: 18-MAR-26
author: MI
purpose: Usa
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |551           |
| :------------ |:-------------|
|**Title**      |Deprecate RSKIP459 |
|**Created**    |18-MAR-26 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP deprecates RSKIP459 [[1]](#references). The Bridge will mark a peg-in transaction as processed only when a fund transfer has been completed: either RBTC is transferred to the receiver (accepted peg-in) or BTC is refunded to the sender (rejected but refundable). Rejected, unrefundable peg-ins will not be marked as processed.

## Motivation

Under RSKIP459 [[1]](#references), the Bridge marks rejected, unrefundable peg-ins as processed so that rejection events are emitted only once. That approach requires the Bridge to store and track these transactions even though no fund transfer occurs—neither RBTC minted nor BTC refunded. This adds unnecessary entries to Bridge storage for transactions that will never result in any fund movement.

A more efficient design is for the Powpeg node to filter which transactions it informs the Bridge about. The Powpeg node can determine before contacting the Bridge whether a peg-in would be accepted, rejected but refundable, or rejected and unrefundable. By not informing the Bridge about transactions that are known to be non-processable (e.g. unrefundable rejected peg-ins), the Bridge avoids storing them and only holds state for transactions where a fund transfer has been or will be completed. The Bridge then marks as processed only those peg-ins where funds are actually transferred: RBTC to the receiver (accepted) or BTC refunded to the sender (rejected but refundable). Rejected, unrefundable peg-ins are not marked as processed and, under normal operation, are not submitted to the Bridge by the Powpeg node; re-emission of unrefundable events would only occur if a user manually attempts to register such a transaction.

## Specification

The behaviour specified in RSKIP459 [[1]](#references) is deprecated. The following rules apply instead.

When a peg-in is **rejected and unrefundable**, the Bridge must **not** mark the transaction as processed after emitting the `rejected_pegin` and `unrefundable_pegin` events. The transaction may therefore be re-registered and the events re-emitted on later processing attempts.

When a peg-in is **accepted**, the transaction is marked as processed (unchanged). When a peg-in is **rejected but refundable**, the transaction is marked as processed after the refund (unchanged).

In summary:

| Outcome | Fund transfer | Mark as processed |
|--------|----------------|-------------------|
| Accepted peg-in | RBTC transferred to receiver | Yes |
| Rejected, refundable | BTC refunded to sender | Yes |
| Rejected, unrefundable | None | No |

### Powpeg node behaviour

The Powpeg node should be updated so that it does **not** inform the Bridge about transactions that are known to be non-processable (e.g. peg-ins that would be rejected and unrefundable). Under normal operation, the Bridge would therefore only receive such transactions if a user manually attempts to register them. As a result, the emission of unrefundable events and any duplication would in practice occur only in that manual-registration scenario.

## Rationale

### Semantic consistency

"Processed" is defined as "the Bridge has completed a fund transfer for this transaction." Unrefundable rejections involve no such transfer, so they should not be marked processed. This keeps one clear meaning and avoids overloading the flag for downstream systems that interpret "processed" as "funds have been moved."

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [RSKIP459](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP459.md): Mark rejected peg-ins as processed

[2] [RSKIP181](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP181.md): Peg-in rejection events

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
