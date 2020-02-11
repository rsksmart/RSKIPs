# Web3 getBlock cumulativeDifficulty

|RSKIP          |157           |
| :------------ |:-------------|
|**Title**      |Web3 getBlock cumulativeDifficulty|
|**Created**    |11-FEB-20 |
|**Author**     |MP |
|**Purpose**    |USa |
|**Layer**      |Node |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

The current RSKIP proposes a new field `cumulativeDifficulty` to be added to the response of existing Web3 Block querying methods. Such field consists in the sum of a Block's difficulty plus its uncles' difficulties.

## Motivation

We define the `difficulty` as a magnitude indicating the effort required _for a single miner_ to mine a single block for it to be added to the blockchain. However, this single value does not represent the total effort put by _all miners_ in the network, since the block not only anchors to its parent, but to its uncles as well. We thus define the magnitude `cumulativeDifficulty`, which encompasses not only the block's difficulty, but its mined uncles' difficulty as well.

Due to multiple needs such as forensics or statistics, it may sometimes be needed to make a reconstruction of the network hashrate evolution throughout a certain period of the blockchain, if not all. Since the hashrate must consider not only the mainchain block, but uncles as well, it must be computed using the above defined `cumulativeDifficulty` magnitude, since it entails the effort performed by every participant of the network and not only that of whomever achieved to mine the winning block.

Since the current Web3 standard does not offer a method to obtain the block and its uncles full information, performing this reconstruction involves multiple interactions with the node. Thus, we propose adding a `cumulativeDifficulty` field to the existing `eth_getBlockX` and `eth_getUncleX` responses. 

## Specification

The field `cumulativeDifficulty` is to be added to the following Web3 methods:

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
