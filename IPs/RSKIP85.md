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
Process and calculate rewards for the current block reading only block data from the blockchain
Avoid storing any new data related to blocks on the state storage
Be able to pay rewards in an efficient way, without using RLP encoding and decoding.
Process fees and rewards only when it’s meaningful for the network

# Specification

In this new implementation REMASC computes the fees by accessing data directly from the block headers instead of using its own storage. This involves three changes:

Increase the size of the internal block cache, so required blocks to calculate rewards are present in a cache in local memory with high probability. 
Remasc storage will hold no uncle information as this will be retrieved from the blockchain.
A clean up of REMASC store will be done. The siblings data will be erased, and won’t be stored anymore
Fees will be processed only when a minimum value is reached.

Item 1 only changes the source of the information, but items 2, 3 and 4 can only be implemented through a hard fork.
So, starting on block N:

REMASC will stop reading its storage for processing fees and will read it from the block headers instead. This involves all necessary information to calculate the mining reward and if a REMASC selection rule was broken. 

A new rule is added. Minimum payable gas is defined as 21000. Mining reward is paid only if a minimum threshold of minimum payable gas * minimum gas price is exceeded. If the threshold is not satisfied the full block reward is not distributed and added back to the reward balance.

All siblings information in REMASC is erased.

# Rationale

REMASC currently stores a lot of redundant data which can be retrieved from the headers of the ancestor blocks. Also all the uncles are stored in a single cell which is only partially updated during block processing, causing high CPU and disk consumption. The first one when encoding and decoding rlp data, and the second one it’s because REMASC reserializes this kind of cells multiple times.
The main goal for this improvement is to reduce the storage used by REMASC without any harm to cpu usage. At the same time, this proposition opens the door to some more improvements. For example, current lookup of blocks to process is linear and it can be improved to O(1) in average case and O(log n) in the worst case.

The value of 21000 gas for the minimum payable gas was added to make payments meaningful for the state of the blockchain at any time. Payments update storage values and consume processing time, so it was considered that a minimum value was required.

Some alternatives were considered, the first option was storing only the specific data about the uncles and the coinbase. This was interesting since the node would only need space on memory for this specific fields instead of the whole block. This alternative wasn’t picked since the information stored on the blockstore and the blockcache inside could be used for other purposes. Extending the current blockcache would enable to resolve other features in the node in a more efficient way by making available more data about blocks on memory.

# Backwards Compatibility

This change should be introduced on a network upgrade. When the marked block is reached the change on the implementation will be reflected and the storage for the REMASC contract is going to stop erasing and saving blocks data. This will produce a great improvement for the disk usage of a RSK node.

# Performance

With the changes explained on this document we expect to see an improvement on block processing time. Whenever a block doesn’t reach the minimum threshold, no rewards will be paid avoiding making changes on all the accounts involved for rewards. This will have impact only on some blocks since we expect that the minimum threshold will be surpassed on most blocks. However the other change introduced in REMASC will have a greater impact since it will avoid reading and writing a large amount of data from the disk. Now all this data on the average case will be read from the blockstore cache and won’t need to be stored anymore.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).