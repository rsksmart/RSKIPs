---
rskip: 224
title: Include Uncles in CPV in Fork Detection Data
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-04-01
---
# Include Uncles in CPV in Fork Detection Data

|RSKIP          |224           |
| :------------ |:-------------|
|**Title**      |Include Uncles in CPV in Fork Detection Data|
|**Created**    |1-APR-2021 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     |https://research.rsk.dev/t/rskip-224-include-uncles-in-cpv-in-fork-detection-data/149|

# **Abstract**

This RSKIP proposes an improvement to the Fork Detection Data present in merge-mining tags (used by the Armadillo Fork Detection System). The commits-to-parent vector is replaced by a Commit-to-FamilyMembers vector, where the members are the parents and all uncles previously references, ordered by  reference block number.

# **Motivation**

The RSK merge-mining tags cointains the following fields:

* 7-byte Commit-to-Parents-Vector (CPV)
* 1-byte number of uncles in the last 32 blocks (NU)
* 4-byte Block Number (BN)

The last 3 fields are the Fork Detection Data, as specified in [RSKIP110](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP110.md).

The CPV specifies references to parent blocks, with a 64 block spacing. The CPV allows Armadillo to connect the blocks of potential forks with high cheating cost. However, a malicious miner can privately mine up to 10 uncle blocks per mainchain block, and build a fork with all these uncle references.

The consequence is that the evidence left on the Btcoin blockchain becomes less accurate to predict an attack. 
Now we present an scenario that provides useful information of the effectiveness of Armadillo in such case. We assume:
* an attacker that has 10% more hashing power than the honest chain.
* the attacker follows the strategy of mining the maximum possible number of uncles per block. 
* both the attacker's fork and the honest fork have stable block difficulties.

The attacker can mine privately a maximum of 63*11=693 family members until a divergence between the honest chain and a malicious forks is detected by the CPV. As the attacker has more hashrate than the honest chain, the attacker can perform a double-spend.

A better attack (for the attacker) is to let the difficulty increase in his blocks: this can be done by manipulating the block timestamp. This leads to a fork with approximately 10% more cumulative work when reaching the divergence point.

To improve the accuracy of the CPV, we propose that uncle blocks are counted in the CPV references.


# **Specification**

The new merge-mining tag format is the following:

* PREFIX: 20-byte prefix of the hash-for-merge-mining
* CPV: 7-byte Commit-to-Parents-Vector (CPV)
* NU: 1-byte number of uncles in the last 32 blocks (NU)
* VER_BN4: packed field
* BN:  3-byte Block Number

This is the same base format specified in [RSKIP223](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP223.md).

The field VER_BN4 contains two nibbles. The hi nibble contains the tag version.
The low nibble containst the highest 4 bits of the BN field.

Tag version 0 represents the old tag format and is no longer valid after this RSKIP is activated. Version 1 is reserved for RSK223-only tags if that RSKIP is activated. This RSKIP mandates the use of version 2 for tags compliant with RSKIP224 but not with RSKIP223, and version 3 for tags compliant with RSKIP224 and RSKIP223.

If the BN field overflows, it wraps around zero.

For versions 2 and 3 of the tag, the field CPV is reintepreted as Commit-to-FamilyMember vector (CFV), and contains a reference to a previous mainchain block or uncle, linearly ordered.

Let's assume that a block at height i starts with a family length f(i) and the block references U uncle blocks. Then the mainchain block is assigned the family member index f(i), and an uncle at position j (starting from j=0) takes the family member index f(i)+1+j. We set r(i+1)=r(i)+1+U.

We don't mandate how the node should track the family length at each block. The length can be tracked implicitely by the node in the same way the node tracks the cumulative difficulty of a block, or it can be made explicit in the block header with another consensus change.

Let's assume the consesus change is activated a block number A. The block positioned at height (N-512) will be considered the starting point for counting family members, and it will have a family length of zero f(N-512)=0. The node should start tracking the family length at height A.

Similar to CPV, the CFV field is a vector V with elements v(j) (j = 0..6) where each v(j) consist of the the least significant byte (LSB) of the Bitcoin Block ID associated with the family member (parent or uncle) at height ( [(f(i)-1)/64]*64 - j*64 ). ([x] means x rounded down), where i is the block height.


# Rationale

This change has no impact on the mining pool softwares, as it preserves the merge-mining tag length.

A drawback of this proposal is that nodes must build and cache the last 512 family members to allow fast lookup by member index. There are several different ossible implementation (each with distinct trade-offs) to allow efficient look up of family members by index:

1. A cache can be built incrementally as new blocks arrive and older blocks are discarded, but the cache must be invalidated on forks. The cache is a map of the family member index to the block object.

2. An LSB-cache is built for every block as an array of the 512 Bitcoin block LSBs corresponding to last 512 family members. When a block is processed, the LSB-cache of the previous block is first cloned, then modified according to the new block content, and finally stored in memory along the block or in a separate map by block hash. This cache of LSB-caches would provide constant time access when switching forks. The memory cost becomes at most 512*512 bytes, or 262 Kbytes.

3. The LSB-cache can be stored in a cell of a fixed contract in the world state. Switching between forks would automatically switch the LSB-cache.

4. The  world state could store only the LSBs of the uncles of the current block (the tail of the LSB-cache). Swiching between forks would involve scanning the last (at most) 512 block world states collecting 512 LSBs, thus rebuilding the cache.

We suggest doing (1) or (2) as (3) and (4) require more coinsensus changes.

One possibility to redcue the node blockchain housekeeping is that the block header includes not only the block height, but also the family length, so the node doesn't need to track it. This functionality could be specified in another RSKIP. 

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers that are always in sync do not need to be updated. 

# Test Cases

TBD

## Security Considerations

This change improves the security of the network by allowing more accurate detection of forks.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
