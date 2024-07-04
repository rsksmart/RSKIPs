---
rskip: 434
title: Bridge Bitcoin block chainwork up to 12 unsigned bytes
created: 26-Jun-24
author: JD,JZ,MI
purpose: Sec
layer: Core
complexity: 1
status: Draft
description: Change in Bridge Bitcoin storage to accept blocks with chainwork serialized up to 12 unsigned bytes
---

|RSKIP          |434           |
| :------------ |:-------------|
|**Title**      |Bridge Bitcoin block chainwork up to 12 unsigned bytes |
|**Created**    |26-Jun-24 |
|**Author**     |JD,JZ,MI |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

The Rootstock Bridge functions as an SPV (Simplified Payment Verification) node. This means it operates by receiving block headers from the Bitcoin blockchain through external entities known as relayers. The bridge verifies the validity of Bitcoin transactions by requesting users to submit Merkle proofs, which follow Bitcoin's SPV technique. This setup enables the bridge to confirm that funds have been locked on the Bitcoin blockchain without needing to trust the relayers fully, as the verification is done through cryptographic proof.

This mechanism is crucial for maintaining a trust-minimized operation, ensuring that users can independently verify the locking of funds on Bitcoin, which is essential for the security and functionality of cross-chain transactions between Bitcoin and Rootstock.

Chain work in Bitcoin refers to the cumulative computational effort that has been expended to build the blockchain. It is a measure of the total work done by miners to create and validate blocks, ensuring the security and integrity of the blockchain. The chain work is currently stored in a signed 12 bytes element in the Block store, this is due to a previously unknown limitation in the Bitcoinj library.

As of block #849,138 chainWork has reached a value of 800028897aa1dce163ead2cf which exceeds the capacity of the 12-byte storage format.
The fact that the max value was surpassed implies that the node cannot process new Bitcoin blocks.

This RSKIP proposes a workaround to allow processing Blocks from 849,138 onward.

**This RSKIP does not include required changes to support difficulties beyond the 12 bytes.**

For more information about the incident that stopped Rootstock to process Bitcoin blocks go [here](https://blog.rootstock.io/noticia/incident-report-rootstock-peg-in-peg-out-service-outage-on-june-24th/). [1]

## Motivation

Without the capacity of accepting Bitcoin Block headers, the Bridge looses its functionality as an SPV for Bitcoin.

Peg-in operations won't work until this RSKIP is implemented, and peg-outs will only be available until all Bridge UTXOs are consumed.

Extending the capacity for storing higher chain works is the only way to resume the normal operations.

## Specification

The current implementation of the Rootstock client (RSKj) relies on a customized version of the renowned [BitcoinJ](https://bitcoinj.org/) library [2] to store the Bitcoin Block headers, among many other Bitcoin related operations.
This library experienced an issue with the processing of Bitcoin blocks, starting from block [#849,138](https://mempool.space/block/00000000000000000002717c41c14b7865e24af838dbc374fc1952f9cbf7a2ff) [3] as exposed in the issue [#3410](https://github.com/bitcoinj/bitcoinj/issues/3410) in their repository [4].

The core developers of BitcoinJ quickly proposed and released a [fix for the issue](https://github.com/bitcoinj/bitcoinj/pull/3411) [5].

This RSKIP takes this solution and proposes to implement it in the Rootstock client as well.

The Bridge currently stores the chain work by serializing it using a 12-byte _signed_ byte-array. This RSKIP proposes to change the serialization to use a 12-byte _unsigned_ byte-array.
As this serialization won't use the sign bit, this will effectively extend the stored data to use the full 12 bytes.
According to our calculations this extra bit should allow for close to 2 years of blocks before reaching the limit.

When the limit is hit, the node should fail. This means it should stop accepting any Bitcoin block with higher chain work than `0xffffffffffffffffffffffff`.

## Backward Compatibility

This change is a hard fork and therefore all full nodes must be updated.

## Rationale

This RSKIP proposes a quick solution to the present issue. It's a high value low effort proposal.
The proposed change enables Rootstock to continue processing Bitcoin blocks normally while the community proposes a better long term approach to process even higher difficulties in the near future.

The BitcoinJ community is also actively working on a long term solution in issue [#3415](https://github.com/bitcoinj/bitcoinj/issues/3415) [6]

## References

[1] Rootstock incident report https://blog.rootstock.io/noticia/incident-report-rootstock-peg-in-peg-out-service-outage-on-june-24th/

[2] Bitcoinj https://bitcoinj.org/

[3] Bitcoin block #849,138 https://mempool.space/block/00000000000000000002717c41c14b7865e24af838dbc374fc1952f9cbf7a2ff

[4] BitcoinJ reported issue https://github.com/bitcoinj/bitcoinj/issues/3410

[5] BitcoinJ proposed fix https://github.com/bitcoinj/bitcoinj/pull/3411

[6] BitcoinJ long-term solution discussion https://github.com/bitcoinj/bitcoinj/issues/3415

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
