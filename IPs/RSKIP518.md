---
rskip: 518
title: Network Upgrade - Reed
description: 
status: Adopted
purpose: Usa, Sca
author: AE
layer: Core
complexity: 3
created: 2025-06-13
---
# Network Upgrade: Reed

|RSKIP          | 518                        |
| :------------ |:---------------------------|
|**Title**      | Network Upgrade: Reed      |
|**Created**    | 13-JUN-2025                |
|**Author**     | AE                         |
|**Purpose**    | Usa,Sca                    |
|**Layer**      | Core                       |
|**Complexity** | 3                          |
|**Status**     | Adopted                    |

## Abstract

This RSKIP outlines the consensus changes proposed for inclusion in Rootstock’s upcoming network upgrade, code-named Reed. Some of the proposed RSKIPs, if ultimately approved, are intended to activate only on the Rootstock Testnet initially, as part of an incremental rollout strategy that allows for validation in a less critical environment before Mainnet deployment. These RSKIPs are marked with the (Testnet-only) tag below.

## Specification

- Codename: Reed
- Block activations for Reed 8.0.0:
	- Rootstock Mainnet block: 8,052,200
	- Rootstock Testnet block: 6,835,655
- Block activations for Reed 8.1.0:
	- Rootstock Mainnet block: N/A
	- Rootstock Testnet block: TBD

### Included RSKIPs

- [RSKIP-305](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md): Peg-out efficiency improvement (Segwit)
- [RSKIP-516](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP516.md): Precompiled contracts for +/* on Secp256k1

Note: The remaining accepted RSKIPs will be released in a second release on Testnet as explained later in this document. 

### Accepted RSKIPs

- [RSKIP-305](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md): Peg-out efficiency improvement (Segwit)
- [RSKIP-516](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP516.md): Precompiled contracts for +/* on Secp256k1
- [RSKIP-144](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP144.md): (Testnet-only) Parallel Transaction Execution for the Unitrie (*)
- [RSKIP-502](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP502.md): (Testnet-only) Union Bridge Integration: New Methods Added to PowPeg Bridge Contract (*)
- [RSKIP-529](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP529.md): (Testnet-only) New storage cells in Bridge native contract for base and super events info (*)
- [RSKIP-536](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP536.md): (Testnet-only) Additional methods for the BlockHeader precompiled contract (*)

### Rejected RSKIPs

No rejected RSKIPs

### Proposed RSKIPs

- [RSKIP-305](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md): Peg-out efficiency improvement (Segwit)
- [RSKIP-516](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP516.md): Precompiled contracts for +/* on Secp256k1
- [RSKIP-144](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP144.md): (Testnet-only) Parallel Transaction Execution for the Unitrie (*)
- [RSKIP-502](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP502.md): (Testnet-only) Union Bridge Integration: New Methods Added to PowPeg Bridge Contract (*)
- [RSKIP-529](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP529.md): (Testnet-only) New storage cells in Bridge native contract for base and super events info (*)
- [RSKIP-536](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP536.md): (Testnet-only) Additional methods for the BlockHeader precompiled contract (*)

## Two-Phase Rollout Strategy (*)

Reed will be released in two parts, with Testnet-only features included in the second release. Features in this second, Testnet-only release are marked above with (*).

## Timeline

- JUN-13-25: RSKIP created with an initial list of proposed RSKIPs
- JUN-19-25: Added additional proposed RSKIPs
- AUG-07-25: Core Devs Community Call covering all RSKIPs proposed. The upgrade proposal is open for comments until August 22nd.
- AUG-27-25: The upgrade proposal is now closed for comments. Accepted RSKIPs list has been updated.
- SEP-30-25: Reed successfully activated on Mainnet and Testnet
- OCT-30-25: In preparation for the second Reed release, RSKIP 502 has been split into RSKIPs 502, 529, and 536, all of which are required to run Union Bridge on Testnet.

## References

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

