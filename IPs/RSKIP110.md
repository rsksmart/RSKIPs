---
rskip: 110
title: Fork Detection Data in RSKBLOCK tags 
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: 
complexity: 2
created: 2019
---
#  Fork Detection Data in RSKBLOCK tags

| RSKIP          | 110                                  |
| :------------- | :----------------------------------- |
| **Title**      | Fork Detection Data in RSKBLOCK tags |
| **Created**    | 2019                                 |
| **Author**     | SDL                                  |
|                |                                      |
| **Layer**      | Sec                                  |
| **Complexity** |                                      |
| **Status**     | Draft                                |

# Abstract

RSK is merge-mined. Because miners that mine RSK want to earn Bitcoin rewards, blocks that are not sent to the RSK network but were produced while merge-mining RSK are exposed by the RSKBLOCK tags in the coinbase transaction of the Bitcoin block. The reasons why a miner may not send a RSK block to the RSK network even if he has solved it are many: the node can be isolated from the network by an attacker or by mistake, the node can be out of consensus due to a consensus-splitting bug, or the node can be maliciously building a parallel chain in preparation of a double-spend attack. For all of these cases is of utter importance to be able to correctly diagnose the state of the system. This RSKIP presents a modification to the consensus of RSK that adds additional information to RSK merge-mining tags so that users or automated systems can make informed decisions about the network health. Network nodes can also respond autonomously to abnormal situations in a way that protects economic nodes from double-spends, and also broadcast succinct proofs of such state to other nodes of the network. 

# Discussion

This RSKIP proposes using a new RSK RSKBLOK tag format. The new RSKBLOCK tag must be able to provide enough information to recognize if miners are mining on the same chain and detect if a parallel selfish blockchain is being created, with high probability. The diagnosis method need not be exact, since the ability to detect a double-spend attempt to with high probability may be enough to deter such attempt. 

We could, for example, append the list of the last 60 RSK parent block ids to the tag. This solution is expensive in terms of space consumed in the Bitcoin coinbase transaction. Therefore we want to use a tag that is as short as possible. Also changing the size of the tag (currently 32 bytes) is problematic because it requires modifying every implementation of the RSK merge-mining plugins in Mining Pool servers. 

We'll want to build a tag that respects the space constrain and that is still secure under the assumption of rational miners. Economic incentives must deter miners from abusing the new tag format. 

We designed the tag to include the block number, the number of uncles created in the last 32 blocks and the LSBs of the Bitcoin block IDs corresponding to the 7 closest RSK grand-parent blocks with heights h(i) at checkpoints selected by (h(i) % 64) ==0. 

**Protection against a Malicious Miner trying to hide the fact that he's mining al alternate chain by mimicking the LSB values.**

An LSB from the Bitcoin block ID is considered practically random because it's part of the PoW.  The LSBs cannot be chosen by a miner to match a specific value, without brute-force. Brute-forcing the LSB would discard on average 255 valid Bitcoin blocks in the process and therefore hash a very high cost. A malicious miner that wants to fake the LSBs of the n parents commitment must invest at least 256 times more energy than the honest miners. In other words, a malicious Bitcoin mining pool can only fake a RSK chain if RSK has less than 1/256th of Bitcoin hashing rate.  This also means that if the malicious miner tries to double-spend continuously every 64 blocks, he will succeed once every 2^56 attempts on average (about 2200 billion years). 

**The Accuracy of Information in the Tag**

The 64 block spacing is a trade-off between fork detection precision and a security threshold established by the amount of energy required by the attacker. The Bitcoin blocks only provides "fuzzy" RSK fork information. For example, if RSK had 50% of Bitcoin hashing power, only about one Bitcoin block every 40 RSK blocks will be published in Bitcoin, and therefore the 64-block interval for parent references is in the same order as the unavoidable "measuring error". If RSK hashing power decreases to 25% of Bitcoin's, one every 80 RSK blocks would be referenced in Bitcoin. 

**Protection Against a Malicious Miner Trying to Hide the Overlap between Parent Commitments**

Seven references at 64-block spacing represents a maximum of 448 RSK block references, or about 11.2 Bitcoin blocks on average when mining solo. This means that if a malicious RSK merge-miner has less than 1/11.2th of the Bitcoin hashing rate the blocks he produce will not show an overlap in the committed references, which restricts its usefulness for detecting long parallel chains, but it's still useful to detect forks up to 448 RSK blocks. Anyway, 1/11.2  of Bitcoin hashing power is 9%, so this limitation only appears if RSK had less than 9% of Bitcoin hashing rate, but as of March 2019 is has over 35%.

**Protection Against a Malicious Miner Mining Solo to Create False Positives in the Fork Detection Method**

RSK average block interval is about 30 seconds but the actual ratio between RSK PoW and Bitcoin PoW is not 1:20 because of sibling blocks and the difficulty adjustment algorithm in RSK. Therefore a malicious miner creating a selfish chain and having the same hashing power than the honest chain could grow a chain longer than the honest chain (but with the same total difficulty) . The actual total difficulty of the chain is not exposed in the tag and this is a drawback. It's not easily solvable as the cumulative difficulty is currently a 90-bit number (12 bytes). A delta cumulative difficulty could be stored from a checkpoint not closer than 2^16 blocks, consuming only 11 bytes, but yet this is too much space. For instance, if RSK has 25% of Bitcoin's hashing power, and the honest RSK miners created 1 uncle per block on average, and the attacker had also the same power but would not create uncles, then the attacker would manage to squeeze 160 RSK blocks between his Bitcoin blocks. The community could rise an alarm of a miner creating a private chain longer than the honest chain but it would could be a false alarm. That's why we add 1 byte containing the number of uncles in the last 32 block interval in order to better detect solo mining (note that the interval is 32 here, not 64).

**Protection Against a Malicious Miner Mining Uncles to Create a False Negative in  the Fork Detection Method**

The attacker could mine uncles (up to 7 per block) in order to show lower the apparent hashing rate, if only the length of the chain is observed. However the real attacker's hashing power can be inferred from the number of Bitcoin blocks he publishes, but a statistical variance could make this comparison not very accurate. The additional byte containing the number of uncles gives an additional clue on how the miner is creating the chain.

**Protection Against a Malicious Miner Trying to Reuse a Proof-of-Work** 

Using 20 bytes to identify the RSK block is more than enough because the difficulty of finding a pre-image is 2^160 (much higher than the current 2^70 of the RSK PoW). Creating a 160-bit collision on the merge-mining block identifier by birthday attacks using 2^80 time requires 2^80 memory (1,208,925,819,614 Terabytes), and this is currently impractical. Also a 160-bit prefix collision cannot be used to split the RSK blockchain, because RSK block IDs are still 256-bits long. A prefix collision can only be used to assign the same PoW to two different RSK blocks, and in that case those blocks would share the same height, which means that they would also split the same reward. Only if there are other siblings at the same height the attacker would increase its proportion of the reward. Yet in that case mining a sibling requires less effort, even if the difficulty rose 1 million times. In other words, the expected profit for the attacker would still be negative, even if the prefix had been just 140-bits long.

**Protection Against Malicious Miner Trying to Prevent Transactions From Confirming (liveness)**

This system does not have a specific protection against a Bitcoin miner trying to prevent confirmation of high-valued transactions on RSK that are shielded by this method. The miner can create a completely fake tag. However, the consequences of the attack cannot be worse than the attacker building a real selfish chain to perform a double-spend (an attack on blockchain soundness). The detection system can infer the attackers hashing power by the number of Bitcoin blocks produced to detect a minority miner simulating having high RSK hashing power.

Protection Against a Malicious Miner Trying to Create a new Blockchain from a Past Block

This method cannot prevent a miner from creating a blockchain from a past point (e.g. 2000 blocks back), because the the first of the two transactions involved in the double-spend may have already been accepted. However creating a new blockchain branch from a past point is much more difficult than creating the branch from the current block, because the attacker must outcompete the honest branch.

Any attempt to prevent a new correct branch from replacing an old branch because it is much newer would modify the consensus protocol and would bring network partition risks. This is also true even if this could be accurately established by checking the Bitcoin timestamps.

 **Variants**

It's possible to use references to checkpoints that are not equispaced, but at places that are powers of 2 (32,64,128,256,512,1024,2048). The condition to use a checkpoint would be that all checkpoints of lower power have been surpassed. For example, a checkpoint A of value 128 would only be referenced if checkpoints of values 64 and 32 after A have been referenced. The non equispaced system has the benefit that checkpoints do not "scroll", and with only a single block the monitoring system can pinpoint the fork length within a 2X bound. The method does not require to join several block tags in order to detect chains of length higher than 448 RSK blocks. The drawback is that it's limited to forks of size up to 2048, while by joining equispaced tags one can detect forks of arbitrary size (if and only if the attacker has enough hashing power).



 

# Specification

The 32-byte block header hash-for-merge-mining is replaced by a 32-byte byte array with the following format:

- 20-byte prefix of the hash-for-merge-mining (PREFIX)
- 7-byte Commit-to-Parents-Vector (CPV)
- 1-byte number of uncles in the last 32 blocks (NU)
- 4-byte Block Number (BN)

The CPV field is a vector V with elements v(i) (i = 0..6) where each v(i) consist of the the least significant byte (LSB) of the Bitcoin Block ID associated with the RSK block at height ( \[(BN-1)/64\]\*64 - i\*64 ). (\[x\] means x rounded down)

The number of uncles (NU) can be zero if the last 32 blocks do not reference any uncle. The uncles referenced can be siblings of blocks that are prior to the last 32 blocks. The block being mined does not count. Each block can reference a maximum of 10 uncles, so 32 blocks can reference 320 uncles, but he space used by the NU field is only 1 byte. Therefore, if the number of uncles is above 255, the value 255 is stored. The existence of the amount of uncles would indicate an exceptionally abnormal situation of the network or an attack.

The BN field must correspond to the RSK block height that is being mined.

All values must be stored in big endian.

The four fields must be checked in consensus. This system requires nodes that perform state sync to download 448 block headers prior to the block they would like to start validating. 

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).



```

```
