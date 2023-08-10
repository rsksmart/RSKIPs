---
rskip: 374
title: Reestablish the number of block confirmations for a PowPeg migration period
created: 15-FEB-23
author: MI
purpose: Usa,Sec
layer: Core
complexity: 1
status: Adopted
description: 
---

|RSKIP          |374           |
| :------------ |:-------------|
|**Title**      |Reestablish the number of block confirmations for a PowPeg migration period |
|**Created**    |15-FEB-23 |
|**Author**     |MI |
|**Purpose**    |Usa,Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

This RSKIP proposes a change to the number of block confirmations required for a PowPeg migration period.

## Motivation

As described in RSKIP357 [1], as part of the efforts done to solve the peg-outs service incident [2] it was necessary to increase the PowPeg migration period from 10,585 to 172,800 blocks to give enough time for a mining pool to process the non-standard transactions before the migration period finishes.

Once the issue with the peg-outs has been solved, all migration transactions have been mined and the service has been resumed, this RSKIP proposes to restore the PowPeg migration period back to its original value of 10,585 blocks.

## Specification

The number of block confirmations for a PowPeg migration period is changed back to 10,585 blocks. Since this change is a hard fork, the new value should become active only when activated by the network upgrade that includes this change.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [RSKIP 357](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP357.md)


[2] [Incident report: RSK peg-out service outage](https://blog.rsk.co/noticia/incident-report-rsk-peg-out-service-outage/)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
