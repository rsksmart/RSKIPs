---
rskip: 358
title: Network Upgrade (patch) - Hop 4.0.1
description: 
status: Adopted
purpose: Usa, Sec
author: AE
layer: Core
complexity: 2
created: 2022-10-26
---
# Network Upgrade (patch): Hop 4.0.1

|RSKIP          |358           |
| :------------ |:-------------|
|**Title**      |Network Upgrade (patch): Hop 4.0.1 |
|**Created**    |26-OCT-2022 |
|**Author**     |AE |
|**Purpose**    |Usa,Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

## Abstract

This RSKIP specifies the changes included in the RSK patch network upgrade named Hop 4.0.1. The goal of this network upgrade is to fix the network issue that produced the peg-out service outage as described in [1].

## Specification

- Codename: Hop 4.0.1
- Activation:
	- RSK Mainnet block: #4,976,300
	- RSK Testnet block: #3,362,200
- Open for comments until: Dec, 2nd, 2022 (CLOSED).

### Included RSKIPs


- [RSKIP-353](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP353.md): Align RSK P2SH redeem script with Bitcoin Core standard transactions checks
- [RSKIP-357](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP357.md): Adjust the number of block confirmations for a PowPeg migration period

### Accepted RSKIPs

- [RSKIP-353](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP353.md): Align RSK P2SH redeem script with Bitcoin Core standard transactions checks
- [RSKIP-357](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP357.md): Adjust the number of block confirmations for a PowPeg migration period

### Rejected RSKIPs

- No rejected RSKIPs so far

### Proposed RSKIPs

- [RSKIP-353](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP353.md): Align RSK P2SH redeem script with Bitcoin Core standard transactions checks
- [RSKIP-357](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP357.md): Adjust the number of block confirmations for a PowPeg migration period

## Timeline

- Dec-02-22: Open for comments period is now finished. Release and activation dates for the patch will be announced soon.
- Nov-18-22: This RSKIP will be open for comments until Dec 2nd.
- Nov-17-22: RSKIPs 353 and 357 have been included in RSK Hop Testnet 4.0.1/4.1.1 as described [here](https://blog.rsk.co/noticia/rsk-hop-testnet-4-0-1-and-4-1-1-are-here-testnet-only-versions/).
- Oct-26-22: RSKIP created and proposed RSKIPs listed

## References

[1] [Incident report: RSK peg-out service outage](https://blog.rsk.co/noticia/incident-report-rsk-peg-out-service-outage/)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

