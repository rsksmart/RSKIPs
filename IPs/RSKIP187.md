---
rskip: 187
title: Network Upgrade - Iris 
description: 
status: Adopted
purpose: Usa, Sec
author: AE (@adrian.eidelman)
layer: Core
complexity: 2
created: 2020-11-20
---
# Network Upgrade: Iris

|RSKIP          |187           |
| :------------ |:-------------|
|**Title**      |Network Upgrade: Iris |
|**Created**    |20-NOV-2020 |
|**Author**     |AE |
|**Purpose**    |Usa,Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

## Abstract

This RSKIP specifies the changes included in the RSK network upgrade name Iris.

## Specification

- Codename: Iris
- Activation:
	- RSK Mainnet block: 3,614,800
	- RSK Testnet block: 2,060,500

## Included RSKIPs

- [RSKIP 153](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP153.md): Add BLAKE2 compression function `F` precompile
- [RSKIP 169](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP169.md): Rectify EXTCODEHASH implementation
- [RSKIP 170](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP170.md): 2WP peg-in transactions to any address
- [RSKIP 171](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP171.md): Arbitrary-length data return mechanism
- [RSKIP 174](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP174.md): Preserve balance in contract creation
- [RSKIP 176](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP176.md): Trustless fast BTC bridge
- [RSKIP 179](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP179.md): BTC-RSK timestamp linking
- [RSKIP 181](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP181.md): Add 2WP peg-in transactions reject events
- [RSKIP 185](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP185.md): Add 2WP peg-out transactions events and refund support
- [RSKIP 186](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP186.md): Preserve RSK PowPeg activation block height
- [RSKIP 191](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP191.md): Remove non Ethereum opcodes from virtual machine
- [RSKIP 197](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP197.md): Error Handling for Precompiled Contracts
- [RSKIP 199](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP199.md): Bridge performance improvement
- [RSKIP 200](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP200.md): ReceiveHeaders method limitations
- [RSKIP 201](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP201.md): Time-locked Emergency Multisignature
- [RSKIP 218](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP218.md): New fee rewards address for the RSK Core Developers Fund
- [RSKIP 219](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP219.md): Minimum peg-in and peg-out values reduced
- [RSKIP 220](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP220.md): Open Bitcoin blockchain oracle

## Discarded RSKIPs

- [RSKIP 188](https://github.com/rsksmart/RSKIPs/pull/188): Precompile for BLS12-381 curve operations
- [RSKIP 172](https://github.com/rsksmart/RSKIPs/pull/172): Subroutines for the VM

## Timeline

* 2021-07-15: Updated tentative block activation numbers
* 2021-07-08: Updated block activation numbers and missing RSKIPs link
* 2021-03-12: RSKIPs 172 and 188 have been removed since this VM changes are not being included in upcoming Ethereum hard fork
* 2021-01-15: Updated deadline: 2021-02-01 deadline to accept proposals for Iris
* 2020-11-20: RSKIP proposed

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
