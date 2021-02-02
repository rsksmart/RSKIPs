# Ephemeral Calldata using Precompile

|RSKIP          |214           |
| :------------ |:-------------|
|**Title**      |Ephemeral Calldata using Precompile|
|**Created**    |29-JAN-2021 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes that data can be transmitted to contracts such that if the data if not used by contracts for some time, the data is disposed from the blockchain.

# **Motivation**

Many smart contracts perform computations based on information coming from the outside world. This data must be transferred to the contracts in onchain transactions. If the data is long, the cost of such transfers is significant, because data transfers consume several kinds of resourced in a full node.  Every byte transferred:

* consumes bandwidth during transaction propagation

* delays block propagation because it increases the inter-node transfer time

* delays block propagation due to additional computation (when the data must be processed during execution of the transactions in the block at each hop).

* must be stored in the blockchain forever.

* needs to be read from HDD/SSD when requested by a peer

* needs to be stored in RAM during  processing

Each of these factors represent a cost to the network that must be prices in gas. This proposal presents a technique to transfer data to smart contracts at lower gas costs, thus enabling use cases that were previously economically impractical.

# **Specification**

We split the specification in the sections : Transaction Format Extension, Transaction Execution, Ephemeral Calldata Registration, Ephemeral Calldata Retrieval, Node House-keeping and Gas Costs.

## Transaction Format Extension

A new type of transaction version 2 is defined. This can be done according to [RSKIP145](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP145.md), [RSKIP212](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP212.md, [RSKIP213](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP213.md) or another transaction versioning standard chosen by the community. The transaction adds a new field called ephemeralCalldata. Normally, to sign a transaction, the transaction data hash must be computed. However, when transactions version 1 is signed, the field ephemeralCalldata is transformed before hashing to hash not the value, but the hash of the value.  
If this proposal is implemented with RSKIP212 versioning, the calldata field is replaced by the RLP encoding of the RLP tuple (calldata, ephemeralCallDataSize, ephemeralCalldataRoot). The calldata field is renamed **calldataSummary**. The double RLP encoding ensures  that the outer most encoding is still a single element and not a list, preserving hardware wallet compatibility. 
If this proposal is implemented with RSKIP213 versioning, then a new field ephemeralInfo can be added to the transaction containing the tuple phemeralCallDataSize, ephemeralCalldataRoot) or an empty value (if no ephemeral data is present). 
In case of RSKIP145 versioning, the fields ephemeralCallDataSize, ephemeralCalldataRoot are added separately with new field ids. 

## Transaction Execution

The field ephemeralCallDataSize represents the size of the ephemeral calldata, and must not contain padding zeros. This encoding enables the removal of the ephemeral data. The field **ephemeralCalldataRoot** is defined as buildMerkleRoot(ephemeralCalldata) The function **buildMerkleRoot** merkelizes the ephemeral calldata and returns the root hash. The merkelization fills all unused intermediate or leaf nodes with zero data. The maximum size of ephemeralCalldata is 100,000 bytes.

An **ephemeral transaction** is one that holds ephemeral calldata (specifying a non-empty field ephemeralCalldata). The base gas cost for each ephemeral calldata byte in the transaction is 8 gas. 

## Ephemeral Calldata Registration

A new precompiled contract `Context` exists at address `0x00..0100001B`. The contract has several new methods related to handling ephemeral calldata.

A ephemeral transaction can register the ephemeral data by calling (from any sender) the the method `registerEphemeralCalldata(uint id) returns(uint32 err)` of the `Context` precompile, and only if this call does is not reverted. If this method is called by a transaction without ephemeral calldata, it returns an error code 1. On success, it returns 0.
If there is ephemeral calldata and the method `registerEphemeralCalldata()` is not called, the ephemeral calldata is immediately erased and the block containing the transaction does not need to transfer the ephemeral calldata. 
When `registerEphemeralCalldata()` is called, the ephemeralCalldataRoot, the id passed as argument and the block number and the transaction id are stored inside the `Context` contract.  Internally, the `Context` contract uses two dictionaries **epByBlockNumber** and **epBySID**. The acronym SID stands sender and id, and is built as the the keccak hash of those fields concatenated. The dictionary  epByBlockNumber is indexed by block number. Each entry contains another dictionary **epEntryMap**.  The dictionary epEntryMap is indexed by SID and contains the transaction hash, the ephemeralCalldataRoot and the ephemeralCallDataSize. The epBySID  dictionary is indexed by SID and contains the transaction hash, ephemeralCallDataSize,  ephemeralCalldataRoot and block number.

The method `registerEphemeralCalldata()` can be called multiple times by different contracts during the execution of the transaction. Each call can register the same ephemeral calldata under a different SID. If the same sender and id is used more than once, no additional registration will occur, but the method execution gas is consumed anyway.


## Ephemeral Calldata Retrieval

An epoch is an interval of blocks M blocks. Ephemeral calldata will be available for an epoch. We propose M=180,000. Depending on the average number of uncles per block, an epoch can a last from 1 month to approximately 2 months.

The ephemeralCalldataRoot value can be obtained in the transaction the ephemeral calldata is included by calling the method `getCurrentEphemeralCalldataRoot() returns (uint256 rootHash)` of precompiled contract `Context`.

Ephemeral calldata can be retrieved by contracts by calling the methods:

* `getEphemeralCalldataSize(address sender, uint256 id, uint32 blockNumber ) returns (uint32 size)` 
* `getEphemeralCalldata(address sender, uint256 id, uint32 offset,uint32 maxSize) returns(bytes data)`
* `getEphemeralCalldataRoot(address sender, uint256 id) returns (uint256 rootHash)` 

These methods are provided by the precompiled contract `Context`. If the ephemeral calldata is registered at block at height X, then it will be available to be retrieved in the block range [X+1..X+M+1]. The node can store the ephemeral data longer, but at height greater than (X+M+1) it becomes unavailable to smart contracts. If the same data is re-sent in another transaction, and the data is registered under the same id by the same sender, it may last more blocks. In this case the entries in the `Context` dictionaries will be updated. If the ephemeral calldata was not sent, or not registered, or if it became unavailable, then: 

* `getEphemeralCalldataSize()` returns zero 
* `getEphemeralCalldata()` fills the return buffer with zeros up to maxSize
* `getEphemeralCalldataRoot()`  fills 32 bytes of the return buffer with zeros

## Node House-keeping

The `Context` smart-contract also has a method used by the node for house-keeping: `getEphemeralDataPersisted() returns (uint256[] txHashList)`. When called at block height X, this method returns the transaction hashes of transactions containing registered ephemeral calldata that need to be persisted because they have been queried, and that been included in block (X-2M). This method is not available to be called by onchain transactions, to enable backward compatible upgrades to the `Context` contract. The method `getEphemeralCalldataPersisted()` does not report persistent transaction hashes for any other block than the one at height (X-2M). How the node removes the ephemeral calldata from transactions is implementation-dependent. An easy method is that the node calls `getEphemeralCalldataPersisted()` at every block at height X, and scans all transactions in block (X-2M) for transactions containing ephemeral calldata, and updates all transactions having ephemeral calldata for transactions not included in the set returned by `getEphemeralCalldataPersisted()`. A more efficient method would be to store the ephemeral calldata separately.

A block having M confirmations is considered settled and cannot be reverted, and it's called a **settled block**.  A block is **immutable** when it has 2M valid confirmations. A node can remove the unreferenced ephemeral transaction calldata from the local storage once the block becomes immutable. 

## Gas costs

The gas cost of `getEphemeralCalldata()` is 2000 plus an conditional cost C. If the ephemeral calldata was successfully returned before by any contract, C=ephemeralCallDataSize/16. If the ephemeral data is available but it was never retrieved, then C=ephemeralCallDataSize*60. Therefore `getEphemeralCalldata()` maximum gas cost is 602,000.

The method `getEphemeralCalldataRoot()` can be used by smart-contracts to to verify fraud proofs specified with Merkle paths in the ephemeral calldata without retrieving the calldata itself, which reduces the cost of fraud proves, making them affordable to most users. The gas cost of  `getEphemeralCalldataRoot()` is 2000.

The gas cost of `getEphemeralCalldataSize()` is 2000.

The gas cost of `registerEphemeralCalldata()` is 10000.


# Rationale

The ephemeral data was designed to achieve three properties: Segregated, Conditionally-stored and Delayed.

**Segregated**

Ephemeral calldata is not directly hashed along with the other transaction data to be signed, only the hash digest of the Ephemeral calldata is. Therefore  ephemeral calldata can be disposed, and only the hash digest and its size remains in the blockchain. This pattern is sometimes used for witness data in blockchains.

**Conditionally-Stored**

Ephemeral calldata only becomes part of the blockchain if it is retrieved by a contract using the query methods. If it is not used before a deadline, the data is removed from the blockchain. Full nodes can be optimized to store ephemeral calldata in efficient databases that enable removal without re-packing.

**Delayed**

Ephemeral calldata is delayed one block. A smart-contract can only access the ephemeral calldata starting from the following block of ephemeral calldata inclusion. This reduces the stress to process the ephemeral data in the critical block propagation path from miner to miner. The ephmeral calldata can be sent after the propagation of the standard block data, so that ephemeral data transmission occurs in parallel of block validation at the destination node. Even if there is a risk that a miner creating a block does not provide the ephemeral calldata after the block, forcing the destination peer to backtrack, this is the same risk that a node undergoes when processing a block full of transactions in case the last transaction is invalid. 


## Ephemeral Calldata Use Cases

Ephemeral calldata has many use cases. We describe two of them.

**Optimistic Rollups**

A Rollup is a blockchain scaling technique that moves transaction execution offchain, while keeping the transaction data onchain. The rollup operator performs periodic commitments on the transaction data and ledger state after the transactions were performed. 
Operators pre-deposit security bonds that are slashed in case they misbehave by presenting transaction data that does not lead to the committed state. Generally users do not need to inspect the transaction data, and they can trust the commitment, as long as there are other systems (generally called watchtowers) that are checking the correctness of the commitments and can alert of fraud through **fraud proofs**. Fraud proofs can only be presented for a period after the operator commits to a state. A rollup transaction has two components: the description and the signature. The transaction description generally must be preserved for users to be able to reconstruct the rollup ledger state in case the operator fails (the exception is if the full ledger is checkpointed in the blockchain). However, after some time, there is no need to store the signatures anymore, because fraud profs for those transactions are not accepted.
There are two types of optimistic rollups: those that post onchain an aggregated signature for all transactions (either a multi-signature or a batched signature) and those that post onchain one signature per transaction. In all cases of aggregation, the signature occupies a short space, and ephemeral data is not suitable for signatures. If there is no aggregation, then there is a huge benefit to send all the signatures as ephemeral calldata in a few transactions. Without aggregation signatures can represent from 50% to 85% of the transaction data depending on the compression and signature scheme used in practice.

Suppose that all the ephemeral data is used to store BLS signatures of a rollup system. We assume the block has a 8M gas limit, and that 50% of the space is used to store ephemeral data. Therefore a block can store up to 500 kilobytes of ephemeral data, split into 5 transactions. The block can pack 15k signatures (assuming 32 bytes/signature). If the average block rate is 30 seconds, this implies the transaction procesing of the rollup can reach 500 tps. The remaining block space can be used to store the transaction data for a rollup.

**Fair Multi-player Games**

When the number of participants taking part of a multi party protocol is high, the risk of communication interruptions rises considerably. Imagine a fair multi-party game with the following properties:

- The participants do not trust each other
- Each participant must act at frequent intervals
- The time for a participant to act is bounded 
- Each action can be described by a binary data string
- Onchain verification of an action is very expensive in terms of gas 
- Participants exchange their actions offchain
- Each participant verifies all other actions

This use case is closely related to state channels. Clearly, if a participant does respond to offchain communications on time, the remaining participants can request to the unresponsive participant to post his actions as onchain data with an "data availability challenge", so they can  continue playing. The cryptocurrency network is used as a secure broadcast medium on-demand. However, a participant can falsely accuse another from being unresponsibe and challenge him to post the action data onchain. 
The participant defending has no other option than to post the calldata, which is generally expensive in terms of gas.
If action data is posted as ephemeral calldata, the fairness of the system is increased because responding to a data availability challenge becomes a lot cheaper. Allowing action data to be sent as ephemeral calldata is particularly useful when action data is large (increases the cost of posting the data) or the number of participants is high (increases the chances of broken communications). 


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).