---
rskip: 56
title: Sporadic Verification-less mining
description: 
status: Draft
purpose: Fair
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2017-03-11
---

# Sporadic Verification-less mining

|RSKIP          |56           |
| :------------ |:-------------|
|**Title**      |Sporadic Verification-less mining|
|**Created**    |11-MAR-2017 |
|**Author**     |SDL |
|**Purpose**    |Fair |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |

# **Abstract**

This RSKIP defines how miners can mine a child block without verifying the parent block, but imposes that the child block to include none or less transactions.

# **Motivation**

Verification-less mining allows miners to start creating a child block as fast as they receive a parent block header. If such block turns out to be invalid, then they can revert after they discover this or after S seconds, whatever comes first. Verification-less mining is important in RSK because block interval is short, and therefore miners with low hashing power creating large blocks or blocks that consume much CPU to verify have higher probability of creating stale blocks and receiving lower rewards. In the worse case of a well-working blockchain, an uncle is rewarded 50% of that of a normal block. This RSKIP proposes a version of verification-less mining that enable miners to add some transactions to a verification-less block, but not to fill the gas limit to maximize the reward.

## Discussion

[RSKIP8] presents a version of verification-less mining that enables transaction inclusion, but has the drawback that it discourages the inclusion of transactions that execute code.

In this RSKIP, to create verification-less blocks, the state root and transaction roots are replaced by empty fields. This marks a verification-less block.

This means that SPV wallets cannot always obtain the state after a single confirmation: at least two confirmations may be required to detect a new state.

# **Specification**

The block header is changed according to these rules:

1. If the receiptHash and worldstateRootHash are null, then the block is empty.

2. All REMASC related executions are in the block following an empty block.

3. There can’t be more than one empty block in a row. This is the sporadic condition.

The sporadic condition prevents all miners from doing verification-less mining. Only when a block is received very close to its parent, it will be the case that neight the received block nor the parent are fully executed. This is the case when verification-less mining will be triggered. Also miners could use verification-less mining if the recently received best block has not been fully executed. If miners abuse of this feature, half of the blocks will be verification-less, increasing doubling the "real" confirmation time, and reducing the network throughput between up to a maximum of 50%.  This is not as bad as it seems, because it the same hashing power would have been mining uncles.

[RSKIP8]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP08.md		

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).