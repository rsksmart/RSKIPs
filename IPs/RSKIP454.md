---
rskip: 454
title: Add support to bitcoin blocks with chain work up to 32 unsigned bytes
created: 25-Oct-24
author: NC
purpose: Sca
layer: Core 
complexity: 1
status: Draft
description: 
---

|RSKIP          | 454                                                                   |
| :------------ |:----------------------------------------------------------------------|
|**Title**      | Add support to bitcoin blocks with chain work up to 32 unsigned bytes |
|**Created**    | 25-Oct-24                                                             |
|**Author**     | NC                                                                    |
|**Purpose**    | Sca                                                                   |
|**Layer**      | Core                                                                  |
|**Complexity** | 1                                                                     |
|**Status**     | Draft                                                                 |

## Abstract

This RSKIP proposes enabling RSK to accept Bitcoin blocks with chain work up to 32 unsigned bytes(256 bits). This change is necessary to ensure the bridge can process Bitcoin blocks with chain work values that exceed the current 12 unsigned bytes limit(96 bits).

Before [RSKIP434](RSKIP435.md) this limitation was even lower, 12 signed bytes, 1 bit reserved for the sign and 95 bits used to store the chain work value. This 95 bits limit chain work got surpassed by the block #849,138 causing rsk nodes to be unable to process Bitcoin blocks, resulting in a [Rootstock peg-in / peg-out service outage](https://blog.rootstock.io/noticia/incident-report-rootstock-peg-in-peg-out-service-outage-on-june-24th/).[1]

Therefore, in order to guarantee the future stability of the Rootstock network, it is necessary to increase the chain work limit before the chain work value exceed the current 12 unsigned byte limit.

## Motivation

Ensure rsk nodes stability and prevent future disruptions in the Rootstock network caused by the inability to process Bitcoin blocks with chain work values that exceed the current 12-byte limit. 

## Specification

- Store chain work values of Bitcoin block headers using 32 unsigned bytes. This implies that after this RSKIP gets activated, stored blocks in rsk nodes' storage will be 20 bytes larger in size than before the RSKIP activation.
- Add support to serialize and deserialize chain work values up to 32 unsigned bytes of Bitcoin block headers stored in the Rootstock Bridge.
- Deprecate binary checkpoint support in favor of adding support for the textual checkpoint format. Checkpoints with 32-byte chain work must be in textual format to be processed, as the binary checkpoint format does not support mixed Bitcoin header sizes of 12 bytes and 32 bytes for chain work.

### Backward Compatibility

This change is a hard fork, as it increases the size for storing block headers by 20 bytes to support larger chain work values (up to 32 unsigned bytes). Therefore, all nodes must be updated.

## Rationale

Given the continued increase in Bitcoin's network difficulty, the chain work value of Bitcoin blocks will continue to grow and will inevitably exceed 96 bits, which would again cause an overflow issue. Therefore, this long-term solution is needed to prevent future disruption in the Rootstock network.

## References

[1] Rootstock incident report https://blog.rootstock.io/noticia/incident-report-rootstock-peg-in-peg-out-service-outage-on-june-24th/

[2] Bitcoin block #849,138 https://mempool.space/block/00000000000000000002717c41c14b7865e24af838dbc374fc1952f9cbf7a2ff

[3] BitcoinJ long-term solution https://github.com/bitcoinj/bitcoinj/pull/3425/, https://github.com/bitcoinj/bitcoinj/pull/3426, https://github.com/bitcoinj/bitcoinj/pull/3427, https://github.com/bitcoinj/bitcoinj/pull/3428/

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
