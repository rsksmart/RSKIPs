---
rskip: 180
title: Limit the RSK merged mining merkle proof size
description: 
status: Draft
purpose: Sec
author: VK <volodymyr@iovlabs.org>
layer: Core
complexity: 1
created: 2021-07-15
---

# Limit the RSK merged mining merkle proof


|RSKIP          | 180 |
| :------------ |:-------------|
|**Title**      |Limit the RSK merged mining merkle proof size|
|**Created**    |15-JUL-2021 |
|**Author**     |VK |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |


# **Abstract**

In this RSKIP we propose to limit the size of the bitcoin merged mining merkle proof field in a RSK block header.
This check should happen during the block validation step.

## Motivation

RSK merge mining is compatible with the bitcoin protocol. This is: a miner can mine both bitcoin and RSK simultaneously.
But they are not required to do so. They can mine RSK blocks with an invalid BTC header.

In this case, the RSK merge mining merkle proof inclusion can be arbitrarily large. In particular, the merkle tree can have any height.
And while this is the case, it is unreal for a BTC block to have a height higher than 30.
The HSM assumes that the merkle proof is going to be small.

The attackers could use this property to generate a block that's valid for RSKj but not for the HSM by creating a RSK block with a proof of work that has too many intermediate nodes.
This would break the validity chain of the HSM, not allowing it to sign transactions.

## Specification

The HSM expects that the size of the bitcoin merged mining merkle proof field is not larger than ***960*** bytes.
So, there should be the same limit in the RSKj node as part of the consensus protocol.

The *merge mining fields* are the following:

```
bitcoinMergedMiningHeader,
bitcoinMergedMiningMerkleProof,
bitcoinMergedMiningCoinbaseTransaction
```

Out of those three, only the size of the `bitcoinMergedMiningMerkleProof` field should be verified.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated.

## Test Cases

1. Generate a bitcoin block with a merkle proof lower or equal to ***960*** bytes, and an appropriate RSK block with this proof.
   The RSKj node should accept such RSK block while connecting it to the node's chain.

2. Generate a bitcoin block with a merkle proof larger than ***960*** bytes, and an appropriate RSK block with this proof.
   The RSKj node should reject such RSK block while connecting it to the node's chain.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,
