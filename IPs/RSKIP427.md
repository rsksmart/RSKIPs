---
rskip: 427
title: Express the amount value in wei for peg-out related events
created: 17-APR-24
author: MI
purpose: Usa
layer: Core
complexity: 1
status: Adopted
description: 
---

|RSKIP          |427           |
| :------------ |:-------------|
|**Title**      |Express the amount value in wei for peg-out related events |
|**Created**    |17-APR-24 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

This RSKIP proposes a change in the unit used in `amount` field of `release_request_received` and `release_request_rejected` Bridge events. The proposal is to express the amount in **weis** instead of **satoshis**.

## Motivation

According to the original specification [1] `release_request_received` and `release_request_rejected` Bridge events should express the _amount_ field in weis.

## Specification

When `release_request_received` or `release_request_rejected` Bridge events are emitted, the _amount_ field should have the value in weis that the user sent to the Bridge. The change only affects the value used when the events are emitted, signatures remain unchanged.

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [RSKIP185](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP185.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
