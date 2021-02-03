# Ephemeral Blockchain 

|RSKIP          |xxx           |
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

The blockchain serves two puposes:

* Enables nodes to detect incoming transactions.

* Enables nodes to reach consensus. 


The first purpose can be partially achieved by state synchronization. While all transaction data is not available, the most important information, the user account balance, is part of the state.

The second purpose can also be achieved if all nodes start synchronization from a certain old block checkpoint, for which the state is available.

The benefit of performing state synchronization starting from a state checkpoint and apply transactions afterwards is that, under most use cases for the  blockchain, transaction information is much larger than state information. In particular, the amount of state created by computation is much lower than the state created by direct data transfer from transaction calldata.

By starting from a checkpoint, we reduce: 
 
* bandwidth consumed for old block propagation 
* storage costs for storing old blocks
* SSD accesses required for reading the requested old blocks when queried by a peer, reducing the SSD capacity to manage the new block processing workload.

Since bandwidth, storage and I/O accesses are priced in transaction costs, by reducing the consumption of these resources we are able to reduce the transaction cost, thus enabling use cases that were previously economically impractical.

# **Specification**

We call an interval of M=1,000,000 blocks an epoch. This is approximately one year of blocks. An epoch i starts at block height M*i.
Nodes store the world state at the current block N and a checkpoint state at height M*(N/M-1), where '/' is the integer division. The checkpoint state must be available for peers to be queried. Peers will query the checkpointed start during the initial synchronization.
The checkpoint i at block height i*M is created at block (i+1)*M. 
The ckeckpoint i can be removed from storage just before the checkpoint (i+2) is to be created.
 
A block having M confirmations is considered settled and cannot be reverted, and it's called a **settled block**.  A block is **disposable** when it has 2M valid confirmations. 

## Synchronization

When synchronizing, nodes first check the latest block available (height N), then derive from this block number the checkpoint to use (height M*(N/M-1)), then ask for the checkpointed block, then fill all intermediate block headers to reach the tip (or a block close to the tip), then download the checkpointed state, then process all transactions in the downloaded blocks starting in the checkpointed state.
 
## Gas costs

The basic gas cost of a transaction is reduced from 21,000 to 16,000.
The gas per non-zero calldata byte is reduced from 68 to 16. Gas cost of zero bytes is unchanged. 


# Rationale

An Ephemeral blockchain can help many of the state-of-the-art blockchain scaling solutions. Cheap calldata can provide the data availability guarantee required for 2nd layer rollups to reach higher transaction volumes.

RSK currently has more than 50% of Bitcoin merge-mining hashrate. One year of cumulative work in RSK, while having the Armadillo monitoring for parallel forks, is enough to make re-mining impossible without community awareness. 

In the case of the Bitcoin blockchain, there is supporting historical evidence that one year of cumulative proof of work was enough to protect it from genesis to 2020. During this period Bitcoin reached a 200B market capitalization, and the cumulative work required to rewrite the whole Bitcoin blockchain (based on the maximum hashrate at the time) was lower than the cumulative work of 365 days of blocks (the data is shown in the graph "proof-of-work equivalent days" in http://bitcoin.sipa.be/). From 2020 to February 2021 this measure has been increasing up to 600 days, but also the market capitalization reached 600B. 

Finally, the security model of full ephemeral blocks is similar to a drivechain, and RSK is expected to transition to a drivechain or a hybrid Powpeg-drivechain when the drivechain is available in bitcoin. The drivechain uses a long peg-out interval to assure malicious peg-out attempts are visible to the community, which reseambles the one year of confirmation blocks proposed in this RSKIP, augumented by the Armadillo monitoring system to increase fork awarness.

Related to the gas costs reductions, similar cost reductions were proposed and implemented in Ethereum by [EIP2028](https://eips.ethereum.org/EIPS/eip-2028). EIP2028 argues that a reduction in calldata cost will not affect the consensus security. The same argument can be applied with greater confidence to RSK, because it posses a longer average block time. However, this RSKIP goes beyond that argument and reduces the full node resouces required to process the blockchain to support the reduction in calldata cost.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).