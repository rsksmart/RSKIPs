---
rskip: 85
title: Improvements to REMASC contract
description: 
status: Draft
purpose: Sca
author: LS
layer: Core
complexity: 2
created: 2018-07-11
---
#  **Improvements to REMASC contract**  

| RSKIP          | 85                              |
| :------------- | :-----------------------------  |
| **Title**      | Improvements to REMASC contract |
| **Created**    | 11-JUL-2018                     |
| **Author**     | LS                              |
| **Purpose**    | Sca, Medium                     |
| **Layer**      | Core                            |
| **Complexity** | 2                               |
| **Status**     | Draft                           |

# Abstract

The REMASC contract on the current RSK reference implementation stores large sets of values on the state trie, making the blockchain consume too much space and wasting time encoding and decoding RLP data. To address this issues a different REMASC implementation is proposed, where the contract doesn’t require to save data related to the blockchain. This new implementation reads the blockchain and is able to execute and pay the fees without repeating data between REMASC store and the blockchain. 

# Motivation

The new REMASC implementation intends to accomplish 4 goals:
* Process and calculate rewards for the current block reading only block data from the blockchain
* Avoid storing any new data related to blocks on the state storage
* Be able to pay rewards in an efficient way, without using RLP encoding and decoding.
* Process fees and rewards only when it’s meaningful for the network

# Specification

In this new implementation REMASC computes the fees by accessing data directly from the block headers instead of using its own storage. This involves three changes:
Remasc storage will hold no uncle information as this will be retrieved from the blockchain.
A clean up of REMASC store will be done. The siblings data will be erased, and won’t be stored anymore
Fees will be processed only when a minimum value is reached.

These items can only be implemented through a hard fork.
So, starting on block N:

REMASC will stop reading its storage for processing fees and will read it from the block headers instead. This involves all necessary information to calculate the mining reward and if a REMASC selection rule was broken. 

A new rule is added. Minimum Payable Gas (MPG) is defined as 200000. All distributed rewards (including miner's, RSK Labs, and Federation) are paid only if the amount to be paid (expressed in smart bitcoins) is greater or equal than the MPG multiplied by the block's minimum gas price (BMGP). If the threshold is not reached the the ditributed rewards are not distributed and are returned to the REWARD_BALANCE_KEY. A similar rule applies for federation share of the fees. A special federation reward pool is created. This is simply a balance that is maintained in the REMASC contract storage. All payments to the federation go through this intermediate pool in FEDERATION_BALANCE_KEY. A Minimum Payable Federation Gas is defined as 50000 (MPFG). If the pool balance is greater or equal to the MPFG multiplied by the BMGP, then the full federation pool is distributed. If not, then the pool keeps the federation share for that block. 

All siblings information in REMASC is erased at block N.

# Rationale

REMASC currently stores a lot of redundant data which can be retrieved from the headers of the ancestor blocks. Also all the uncles are stored in a single cell which is only partially updated during block processing, causing high CPU and disk consumption. The high CPU consumption occurs when encoding and decoding rlp data, and the high disk consumption occurs when REMASC reserializes these cells multiple times.
The main goal for this improvement is to reduce the storage used by REMASC without increasing CPU usage. At the same time, this proposition opens the door to some more improvements. For example, current block lookup is linear and it can be improved to O(1) in average case and O(log n) in the worst case.

The value of 200000 gas for the minimum payable gas was defined because lower amounts cause resource consumption that costs more than the actual value transferred. All payments update storage values and consume processing time, so it was considered that a minimum value was an improvement with negligible economic change. The same applies to payments made to federators.

Currently the RSK blockchain protocol allows a node to boostraps from a certain trusted state without retrieving all past blocks, by retrieving the state trie only. However, to be able to continue processing contracts the client must also retrieve the past 256 block headers, because of the BLOCKHASH opcode may request them. The proposal changes this property, because from under the changes proposed a  pruned client would need to store not 256, but the past 4000 headers, including all sibling headers of those blocks referenced as uncles. These headers would be required by the REMASC contract to compute future rewards.

# Backwards Compatibility

This change is not backwards compatible because it implies hard-fork. Old nodes would be disconnected from new nodes immediately afger the first block when the new rules apply is executed.

# Performance

With the changes proposed on this RSKIP we expect to see an improvement on block processing time. while the improvement in efficiency varies depending on the conditions, it has been empirically demostrated to be better in average cases. 
Whenever a block reward doesn’t reach the minimum payment threshold, no rewards will be paid avoiding making changes on all the accounts involved in reward distribution. This has a huge impact during the initial times of the RSK blockchain, when most blocks are empty, but less impact in the future.  Reducing the disk I/O implied by contract storage operations will have a greater impact in the future since it will avoid reading and writing a large amount of data from the disk. If this proposal is imlemented, all required headers will be read from the blockstore cache with high probability.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
