---
rskip: 215
title: Ephemeral Blockchain
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-02-03
---

# Ephemeral Blockchain 

|RSKIP          |215           |
| :------------ |:-------------|
|**Title**      |Ephemeral Blockchain|
|**Created**    |3-FEB-2021 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes that old blocks are removed from the blockchain. While removing blocks from the blockchain is a well known local resource optimization technique known as pruning, removing blocks from all nodes in consensus, as proposed here, is not.  This proposal competes with [RSKIP213](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP213.md) as implementing both techniques simultaneously does not seem to be provide much benefit. 

# **Motivation**

The blockchain serves several purposes:

1. Wallets learn about an account balance
2. Wallets list incoming transactions.
3. Nodes reach consensus. 
4. Humans review all user's transaction history.

Learning about the account balance be partially achieved by state synchronization. While all transaction data is not available, the most important information, the user account balance, is part of the state. 

Listing incoming transactions is generally only important when the user is waiting for payments, and need to distinguish between several incoming payments. Retrieving information of very old transactions is unimportant. 

Reaching consensus can also be achieved if all nodes start synchronization from a certain old block checkpoint, for which the state is available.

Finally, reviewing all transaction history for all users may be useful to collect statistics, and also to increase the confidence on the immutability of the past, but it not a requirement for a blockchain to be useful.

Therefore, the conclusion is that there is no need that the consensus force stating synchronization from the genesis block. The protocol can achieve **checkpoint consensus**. Checkpoint consensus should not be confused with "weak subjectivity", which refers to nodes making individual choices on synchronization checkpoints based on local information (peer reputation).  A [recent paper](https://arxiv.org/pdf/2101.05495.pdf) presents the benefits of checkpoint consensus, but does not address the topic of node synchronization. 

## Checkpoint Consensus

Checkpoint consensus is realized by performing a state synchronization starting from a state checkpoint and applying transactions afterwards.  The benefit of checkpoint consensus is that in most blockchain use cases, transaction information is much larger than state information. In particular, the amount of state created by computation is much lower than the state created by direct data transfer from transaction calldata.

With checkpoint consensus, we reduce: 

* bandwidth consumed for old block propagation 
* storage costs for storing old blocks
* SSD accesses required for reading the requested old blocks when queried by a peer, reducing the SSD capacity to manage the new block processing workload.

Since bandwidth, storage and I/O accesses are priced in transaction costs, by reducing the consumption of these resources we are able to reduce the transaction cost, thus enabling use cases that were previously economically impractical.

# **Specification**

We call an interval of M=1,000,000 blocks an epoch. This is approximately one year of blocks. An epoch i starts at block height C(i)=M\*i.
Nodes store the world state at the current block N and at the checkpoint states at heights C(i) for all checkpoints i which have not been removed, and where C(i)<=(N/M)*M, where "/" is the integer division. The state for checkpoint i is s(i). A checkpoint state must be available for peers to be queried after M confirmations. Peers will query the checkpointed state during the initial synchronization. The state s(i) can be created and stored while synchronizing or created at block C(i+1) by re-processing the blockchain from the previous checkpoint.

A chain segment is defined as a blockchain block interval starting from a checkpoint height C(i) and having exactly M blocks for any non-last segment, or in case of the last segment, having the maximum amount of blocks mined.

A checkpoint state is said to be **removed by consensus** if no node will require this state anymore to reach consensus. The state s(i) can be removed by consensus only a certain **removal points** in the processing of blocks. The removal points for checkpoint i are the blocks at heights C(j) for checkpoints j (j>=i+2). The **removal condition** is the following: a state s(i) is removed by consensus if and only if the sum of cumulative work in the blockchain segments for checkpoints (i+1) to j (including the last unfinished segment) is greater than the blockchain segment for checkpoint i. Since a best chain can only be replaced by another chain with higher cumulative work, a consequence of the removal condition is that the removal of a checkpoint by consensus is irreversible. 

A checkpoint state can be removed by consensus but not yet be removed from storage in nodes. The decision of when to remove a checkpoint state from storage is local to each node. 

## Synchronization

The discovery of the best chain is similar to a non-checkpointed blockchain: the chain with more cumulative work wins. However, finding the checkpoint to start processing transactions requires a bit more work.

To find the starting checkpoint, we use formal specification of the removal condition. Let L be the last checkpoint in the blockchain (which may be incomplete due to unmined blocks). Let w(i) be the **segment cumulative work** in segment i for (0<=i<=L). Let t(i) be the **tail cumulative work**, defined as the sum of w(j) for all j such that (i<=j<=L).  Let the **starting checkpoint** be  j such that j=max( j=0..L-2: t(j)>w(j+1)). The node will only request and execute the transactions for all blocks starting from the starting checkpoint.

This condition assures that a checkpoint will not change if blocks following it have decreased difficulty. The checkpoint will only advance if the tail cumulative difficulty is higher than the cumulative difficulty in segment cumulative difficulty. 

Internally, the node starts by requesting all block headers from genesis to the last block informed by a peer, and then the node asks for the starting checkpoint block, then downloads the checkpointed state, then downloads all block transactions starting from the checkpoint, then processes all blocks starting in the checkpointed state. 

## Gas costs

The basic gas cost of a transaction is reduced from 21,000 to 16,000.
The gas per non-zero calldata byte is reduced from 68 to 16. Gas cost of zero bytes is unchanged. 


# Rationale

The idea of prunning old blocks (originally called "[mini-blockchain](http://cryptonite.info/wiki/index.php?title=Main_Page)") dates back to [2013](https://bitcointalk.org/index.php?topic=215936.0). In the mini-blockchain, removing old blocks was performed to maintain a [finite length](https://web.archive.org/web/20150209130232/http://cryptonite.info/files/mbc-scheme-rev2.pdf) of unprunned blocks. Here we propose an algorithm to select when blocks should be discarded based on a security criteria. The criteria protects the blockchain against block reorganizations when a high proportion of honest hashrate turns dishonest or when hashrate is organically decreasing, keeping the lower bound to the time required to revert the blockchain. The mini-blockchain can only protect from such attack assuming a very rigid difficulty adjustment function, which slows down block creation if the hashrate is organically decreasing. A recent proposal called [CoinPrune](https://arxiv.org/pdf/2004.06911.pdf) uses miners' votes on the last valid snapshot to reaffirm snaptshot authenticity instead of the stronger strict consensus validation proposed here. 
An Ephemeral blockchain can help many of the state-of-the-art blockchain scaling solutions. Cheap calldata can provide the data availability required for 2nd layer rollups to support higher transaction volumes.

## Gas Reductions

Similar gas cost reductions than this proposal were implemented in Ethereum by [EIP2028](https://eips.ethereum.org/EIPS/eip-2028). EIP2028 argues that due the fast propagation of blocks, a reduction in calldata cost will not affect the consensus security. The same argument can be applied to RSK even with greater confidence, because RSK has a longer average block time. However, this RSKIP goes beyond that EIP argument and it actually reduces the full node resources required to process the blockchain to support the reduction in calldata cost.

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers that are always in sync do not need to be updated. 

# Test Cases

TBD

## Security Considerations

We consider the security of an ephemeral blockchain historically by analyzing Bitcoin security over time, and as a comparison with the security assumptions of a drivechain.

## Proof-of-work Equivalent Days

RSK currently has more than 50% of Bitcoin merge-mining hashrate. One year of cumulative work in RSK, while having the Armadillo monitoring for parallel forks, is enough to make re-mining impossible without community awareness. An attempt to steal by a 51% attack by Bitcoin miners will destroy the credibility of Bitcoin and backfire on the miners.

In the case of the Bitcoin blockchain, there is supporting historical evidence that one year of cumulative proof of work was enough to protect it from genesis to year 2020. During this period Bitcoin reached a 200B market capitalization, and the cumulative work required to rewrite the whole Bitcoin blockchain (based on the maximum hashrate at the time) was lower than the cumulative work of 365 days of blocks.  This data is shown in the graph "proof-of-work equivalent days" in http://bitcoin.sipa.be/. From year 2020 to February 2021 this measure has been increasing up to 600 days, but also the market capitalization reached 600B. Therefore we believe 1 year of block confirmations is enough for an ephemeral blockchain based on proof of work, but it's possible to make the number of confirmations variable depending on the hashrate or the number of bitcoins in the peg.

## Ephemeral Blockchain vs Drivechain

The security model of full ephemeral blocks is similar, although not equal to, a drivechain. In a drivechain, the majority of the hashrate mining the sidechain can steal the pegged funds as long as they announce the malicious peg-out transaction in advance, and they confirm the malicious peg-out transaction openly for months using their hashrate. In an ephemeral blockchain, the majority of the sidechain hashrate can build a parallel chain fork starting from the last checkpoint but specifying an invalid malicious state in it. In this state, for example, all coins and tokens are owned by a malicious miner coalition. Afterwards if the malicious miner coalition wants to keep receiving Bitcoin block rewards, then it is forced to publicly expose their private fork by mining Bitcoin blocks with RSK Armadillo tags. These tags reveal the fork to the community. If malicious coalition has 51% of the hashrate, only after one year of mining the fork they can overtake the honest chain. One of the community plans for RSK is the transition of its 2-way peg to a drivechain when the drivechain is available in Bitcoin. If that happens, RSK will already be working under the security assumption of honest majority of the Bitcoin hashrate. The security assumptions for an ephemeral blockchain are similar to the ones of the drivechain. In one sense the risk in a ephemeral blockchain is higher, because the drivechain only puts the bitcoins at risk while the ephemeral blockchain also the tokens created in the RSK platform. However, in another sense the ephemeral blockchain reduces the risk, since RSK hashrate can reach the Bitcoin hashrate using the fork-aware merge-mining technique proposed by “External Confirmation Hashrate” ([RSKIP178](https://github.com/rsksmart/RSKIPs/blob/exthashrate/IPs/RSKIP178.md)), “BTC-RSK timestamp linking” ([RSKIP179](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP179.md)) , and the yet unmerged “Timestamp-adjusted block difficulty transfer”, while the miners supporting a drivechain cannot automatically expand their security to 100% of the hashrate. 

In case RSK transitions to a hybrid Powpeg-drivechain is used, a hashrate honest majority assumption is not needed, and an ephemeral blockchain implies a controlled reduction of the security of the RSK blockchain. 


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
