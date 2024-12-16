---
rskip: 460
title: Ignore non-standard outputs when searching for the witness commitment hash
created: 12-DEC-24
author: MI
purpose: Usa,Sec
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |460           |
| :------------ |:-------------|
|**Title**      |Ignore non-standard outputs when searching for the witness commitment hash |
|**Created**    |12-DEC-24 |
|**Author**     |MI |
|**Purpose**    |Usa,Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

Coinbase transactions may sometimes contain non-standard outputs that cause the Bridge to fail parsing them. When iterating through the outputs of a coinbase transaction in search for the witness commitment hash, non-standard outputs should be ignored and continue searching through the remaining outputs.

## Motivation

When a Bitcoin coinbase transaction is registered in the Bridge via `registerBtcCoinbaseTransaction` method, the provided witness information needs to be validated against the witness commitment hash included in the coinbase transaction **[1]**. To find the witness commitment in the transaction, the Bridge iterates through all the outputs in reverse order until the one containing the witness information is identified. Some of the outputs may have a non-standard format, causing the Bridge to fail parsing them. Those outputs should simply be ignored and move on to the next one until the one containing the witness information is found.

Examples of transactions with non-standard outputs:
- Testnet: https://mempool.space/testnet/tx/ae0a6c774908d3ddd334d40cc385ef1c2ad7e6381a69058114899f5ce567f26c
- Mainnet: https://mempool.space/tx/079e68032d9a4cdb222f464b9966756ccb749633aee678c6a51536b4fc38e29c 

## Specification

When searching for the witness commitment hash in a coinbase transaction, ignore malformed or non-standard outputs and keep iterating through the remaining outputs.

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## References

[1] [BIP141 - Segregated Witness](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
