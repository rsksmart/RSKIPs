---
rskip: 8
title: Verification-less mining
description: This RSKIP describes an implementation of storage rent based on enabling the code to periodically perform an operation to deposit the rent before the due time.
status: Draft
purpose: Fair
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-09-29
---

# Verification-less mining

|RSKIP          |08           |
| :------------ |:-------------|
|**Title**      |Verification-less mining |
|**Created**    |29-SEP-2016 |
|**Author**     |SDL |
|**Purpose**    |Fair |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP defines how miners can mine a child block without verifying the parent block, and proposes a version of verification-less mining that allows miners to include transactions in a block optimistically.  
This RSKIP works best with [RSKIP10]  (Transactions never invalidate blocks).

# **Motivation**

Verification-less mining allows miners to start creating a child block as fast as they receive a parent block header. If such block turns out to be invalid, then they can revert after they discover this or after S seconds, whatever comes first. Verification-less mining is important in RSK because block interval is short, and therefore miners with low hashing power creating large blocks or blocks that consume much CPU to verify have higher probability of creating stale blocks and receiving lower rewards. In the worse case of a well-working blockchain, an uncle is rewarded 50% of that of a normal block. This RSKIP proposes a version of verification-less mining that allows miners to include transactions in a block optimistically, even if prior transactions have not been verified.

## Discussion

To reduce the time to build a block, miners can optimistically choose transactions they think will pay the highest fee based on the advertised gasprice and the gascount. Since the gascount only establishes a maximum (transactions don’t need to consume all of it), then miners may end up building blocks filing much less gas capacity than the available. However miners can always include simple transactions (without code execution), because the full cost of those is fully known and paid, and minimum simple account balances can be computed without execution EVM code.

To allow the inclusion of transactions without transaction execution, fields that relate the output of the block verification are moved to two blocks down the blockchain (receipts root and state root).

Normally miners cannot start working on a block without computing the previous block state. With verification-less mining, when a miner receives a supposedly best block at height N it will usually have computed the state and receipts for the block at height (N-1). Therefore, it can  start creating a block at height (N+1) on top on block N, adding the state/receipts of block N-1.

This means that SPV wallets cannot always verify the correctness of transactions (receipts) state after a single confirmation: at least two confirmations may be required to detect a new state.

Miners can include simple payments because, even if they do not know the world state after execution of the prior block, they can infer the minimum balance of simple accounts. For example, if a miner sees a transaction T in block N, that consumes 10 bitcoins from an account A that had 15, then he knows that in block (N+1) he can include transactions that withdraw at most 5 bitcoins from A.

## Other Competing Proposals

[RSKIP56] presents a version of verification-less mining that does not allow optimistic transaction inclusion.

[RSKIP58] presents a header-first block propagation method that does not require verification-less mining and allows arbitrary transaction inclusion.

# **Specification**

The block header is changed according to these rules:

The receiptHash is moved from the block at height N, to the block at height N+2 and renamed prevBlockReceiptHash
The worldstateRootHash is moved from the block at height N, to the block at height N+2 renamed prevBlockWorldstateRootHash.

Nodes will verify that the sum of all gaslimits in a transaction does not overpass the block gas limit. This is to prevent miners from gambling, adding transactions that run code that they are unsure if they can be executed within the block gas limit.

**Problems**

One of the problem of this proposal is that miners may end up only creating verification-less blocks. For their blocks to collect the highest fees they wouldn't pick transactions that execute code, becaause they can't predict the amount of fees paid. Therefore they will pick either simple transactions or transactions that have low gaslimits. More complex transactions will only be chosen if mining does not produce blocks for a certain delay (e.g. 5 seconds) and the parent block has been fully verified. In this case it is expected that the node would create a new block candidate and mining pools could push new work units to workers.  If processing/broadcasting blocks takes a time comparable to average block interval, then miners will end up only creating verification-less blocks. A potential solution would be to decouple transaction publication from transaction execution, so miners can publish transactions, but not execute them. 

[RSKIP10]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP10.md
[RSKIP56]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP56.md
[RSKIP58]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP58.md		

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).