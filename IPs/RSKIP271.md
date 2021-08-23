# Bridge peg-out Batching 


|          | 271 |
| :------------ |:-------------|
|**Title**      |Bridge peg-out batching|
|**Created**    |AUG-2021 |
|**Author**     | SDL |
|**Purpose**    |Sec, Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

#  **Abstract**

One of the problems of the RSK bridge is that the algorithm used for the creating of peg-out transactions is inefficient, because each peg-out request is served in a different Bitcoin transaction. This can lead to a number of cost, usability, efficiency and security problems. This RSKIP proposes a new algorithm for the Bridge to batch peg-out requests so a single Bitcoin transaction and a low number of UTXOs can serve a high number of peg-out requests. 

## Motivation

The current RSK bridge can suffer from a number of problems described in [RSKIP265](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP265.md). In this RSKIP we're mostly concerned with DoS attacks, and high peg-out costs. The proposed solution is to batch peg-outs. 

## Specification

After receiving a peg-out request, and if now previous request is waiting in the queue, the Bridge contract will wait T=3 hours to create a peg-out transaction. During the waiting time, if more peg-out requests are received, they will be queued so that all of them can be batched in a single peg-out transaction. When the deadline is reached, a peg-out event will be triggered. Under normal usage, the Bridge would create no more than 8 peg-outs events a day. Under a rush hour of peg-outs, the Bridge may need to create more than one peg-out transaction simultaneously per peg-out event to reduce the transaction size, and input count. This peg-out splitting into transactions does not change from the current protocol.

### Peg-out Fees

If [RSKIP272](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP272.md) is not activated, then the peg-out fees will be distributed evenly between all peg-out requests as it is today, and no change is required. 

A new Bridge method `getEstimatedFeesForNextPegOutEvent()`is created so that users can estimate the peg-out fees. This method returns the fees of a peg-out transaction containing (N+2) outputs and 1.5 inputs, where N is the number of peg-outs requests waiting in the queue.  This method will be deprecated if RSKIP272 is activated.


## Rationale

The motives for this RSKIP are well explained in RSKIP265.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Security Considerations

This RSKIP resolves an existing, but not critical, platform vulnerability.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).