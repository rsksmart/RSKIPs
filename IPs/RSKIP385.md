---
rskip: 385
title: Bridge method `getEstimatedFeesForNextPegOutEvent` improvement
created: 12-MAY-23
author: MI
purpose: Usa
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |385           |
| :------------ |:-------------|
|**Title**      |Bridge method `getEstimatedFeesForNextPegOutEvent` improvement |
|**Created**    |12-MAY-23 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes an improvement to how the estimated fee for the next pegout event is calculated in the Bridge contract.

## Motivation

As described in RSKIP271 [1], along with the peg-out batching functionality new methods were added to the Bridge contract. One of these new methods is `getEstimatedFeesForNextPegOutEvent()`, created so that users can estimate the peg-out fees. This method returns the fees of a peg-out transaction containing (N+2) outputs and 2 inputs, where N is the number of peg-outs requests waiting in the queue. (N+2) because it considers one extra output for the change and one extra output for the peg-out the user would potentially include in the queue.

With the current implementation, when 0 peg-out requests are waiting in the queue `getEstimatedFeesForNextPegOutEvent()` always returns 0. This can be misleading to users since it is not the correct value. 

## Specification

When there are 0 peg-out requests in the queue, the method `getEstimatedFeesForNextPegOutEvent()` should return the fees of a peg-out transaction containing 2 outputs (one for the pegout and one for the change) and 2 inputs.

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [RSKIP 271](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
