---
rskip: 357
title: Adjust the number of block confirmations for a PowPeg migration period
created: 25-OCT-22
author: AE
purpose: Usa,Sec
layer: Core
complexity: 1
status: Draft
---

|RSKIP          |357           |
| :------------ |:-------------|
|**Title**      |Adjust the number of block confirmations for a PowPeg migration period |
|**Created**    |25-OCT-22 |
|**Author**     |AE |
|**Purpose**    |Usa,Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes a change to the number of block confirmations required for a PowPeg migration period.

## Motivation

As described in [1][2], the PowPeg redeem scripts need to be aligned with Bitcoin Core standard transactions for these to be confirmed on the Bitcoin network and the RSK peg-out service to be resumed. The updated redeem script will become active after a PowPeg composition change finalizes. When a PowPeg composition change is triggered, the new PowPeg becomes active 18,500 blocks after the process starts (around six days). As part of this process, the bitcoins locked in the PowPeg are automatically migrated from the old PowPeg address to the new one. This migration process completes in 10,585 blocks (around three and a half days). Due to the issue described in [1], for the migration process to complete successfully, a set of non-standard transactions must be included in a Bitcoin block by a Bitcoin mining pool. This RSKIP proposes to increase the migration period from 10,585 to 172,800 blocks (around 60 days) to give enough time for a mining pool to process the non-standard transactions before the migration period finishes. It's important to highlight that the need for non-standard migration transactions to be mined is only needed for this particular occasion. 

## Specification

The value for `fundsMigrationAgeSinceActivationEnd` in [BridgeMainNetConstants.java#L102](https://github.com/rsksmart/rskj/blob/0fbd4bbbfa464d5031f09a622ce34778ac8327c3/rskj-core/src/main/java/co/rsk/config/BridgeMainNetConstants.java#L102) changes from `10585L` to `172_800L`. Since this change is a hard fork, the new value should become active only when activated by the network upgrade that includes this change.

## Backwards Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [Incident report: RSK peg-out service outage](https://blog.rsk.co/noticia/incident-report-rsk-peg-out-service-outage/) 
[2] [RSKIP-353: Align RSK P2SH redeem script with Bitcoin Core standard transactions checks](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP353.md)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
