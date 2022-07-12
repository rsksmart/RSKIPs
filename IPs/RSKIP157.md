---
rskip: 157
title: Cumulative Difficulty in JSON-RPC block responses
description: 
status: Draft
purpose: Usa
author: MP <mpicco@iovlabs.org>
layer: Node
complexity: 1
created: 2020-02-11
---
# Cumulative Difficulty in JSON-RPC block responses

|RSKIP          |157           |
| :------------ |:-------------|
|**Title**      |Cumulative Difficulty in JSON-RPC block responses|
|**Created**    |11-FEB-20 |
|**Author**     |MP |
|**Purpose**    |Usa |
|**Layer**      |Node |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

The current RSKIP proposes `cumulativeDifficulty` as a new field to be added in the responses of existing JSON-RPC block methods. The value of this field is the sum of a block's difficulty plus its uncles' difficulties.

## Motivation

The existing `difficulty` field in responses is defined as a magnitude indicating the effort required _for a single miner_ to mine a single block for it to be added to the blockchain. However, this single value does not represent the total effort required by _all miners_ in the network, since the block not only anchors to its parent, but to its uncles as well. We thus define the magnitude `cumulativeDifficulty`, which encompasses not only the block's difficulty, but its mined uncles' difficulty as well.

Due to multiple needs, such as forensics or statistics, we may sometimes need to make a reconstruction of the network hashrate evolution; in full, or for a subset period. Many of the former mentioned needs require `cumulativeDifficulty` for their calculation because it reflects the effort performed by all the miners (mainchain winning block and uncles).

JSON-RPC block methods do not provide the required information to calculate `cumulativeDifficulty` without requiring multiple interactions (querying for a block and all its uncles). Thus, we propose adding a `cumulativeDifficulty` field to the existing `eth_getBlockX` and `eth_getUncleX` responses.

## Specification

The field `cumulativeDifficulty` is to be added to the following JSON-RPC block methods:

* `eth_getBlockByHash`
* `eth_getBlockByNumber`
* `eth_getUncleByBlockHashAndIndex`
* `eth_getUncleByBlockNumberAndIndex`

The following pseudo-code shows how the field is to be computed:

```
def cumulativeDifficulty(block):
    cumulativeDiff = block.difficulty
    
    for uncle in block.uncles:
        cumulativeDiff += uncle.difficulty

    return cumulativeDiff
```

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
