---
rskip: 141
title: "Network upgrade meta: Wasabi+1"
author: AE, SDL, JR, DLL, MM, JD, PP, JP
type: Meta
status: Draft
created: 2019-09-27
requires: Wasabi
---

## Abstract

This meta-RSKIP specifies the changes included in the Ethereum hardfork name TBD.

## Specification

- Codename: TBD
- Activation: TBD

### Included RSKIPs

- TBD

### Accepted RSKIPs

- Add EXTCODEHASH opcode
- Add CHAINID opcode (ref. [EIP-1344](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1344.md))
- 2WP Locking Cap
- 2WP Onchain Events
- ValidateEthPow precompiled contract

### Tentatively Accepted EIPs

- ECADD, ECMUL and Pairing check precompiled contract
- SELF_BALANCE opcode and SLOAD, BALANCE and EXTCODEHASH cost reduction (ref. [EIP-1884](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1884.md))
- 2WP Segwit Compatibility
- Disable TXINDEX, DUPN, SWAPN, OP_HEADER opcodes
- CalculateMMR precompiled contract
- MMR in block header 
- Reduce difficulty adjustement coefficient between blocks
- Add upper limit for voteFeePerKbChange method in precompiled Bridge contract 

### Proposed EIPs

- Decentralize feePerKb calculation for precompiled Bridge contract
- Costs reductions (ref. [EIP-1108](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1108.md) / [EIP-2028](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2028.md) / [EIP-2200](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2200.md))
- Add Blake2 compression function F precompile (ref. [EIP-152](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-152.md))
- GetRefund precompiled contract (ref. [RSKIP-139](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP139.md))
- Add internal transactions hash in block header
- Add bloom filter hash in block header
- Precompiled Bridge contract events in Solidity format

## Timeline

* 2019-11-01 (Fri) hard deadline to accept proposals for "Wasabi+1"

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
