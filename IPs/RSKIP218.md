---
rskip: 218
title: New Fee Rewards Address for the RSK Core Developers Fund
description: 
status: Accepted
purpose: Usa
author: FJ (@fedejinich)
layer: Core
complexity: 1
created: 2021-03-25
---
# New Fee Rewards Address for the RSK Core Developers Fund

|RSKIP          |218           |
| :------------ |:-------------|
|**Title**      |New Fee Rewards Address for the RSK Core Developers Fund |
|**Created**    |25-03-2021 |
|**Author**     |FJ |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Accepted |

## Abstract

Set a new reward address for the RSK Core Developers Fund

## Specification

REMASC precompiled contract should pay fees at a new address.

When `blockNumber > iris_hardfork` update the fee rewards address:
- Old address: `0x14d3065c8Eb89895f4df12450EC6b130049F8034`
- New address: `0xdcb12179ba4697350f66224c959bdd9c282818df`

## References

[1] RSKIP14 https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP14.md

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
