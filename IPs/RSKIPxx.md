# Ephemeral Calldata using Precompile

|RSKIP          |XXX           |
| :------------ |:-------------|
|**Title**      |Ephemeral Calldata using Precompile|
|**Created**    |29-JAN-2021 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes that data can be transmitted to contracts in a way that its availability is delayed, and if not used for some time, the data is disposed.

# **Motivation**

Most smart contracts perform computations based on information coming from the outside world. This data must be transferred to the platform through the transaction layer. The cost of such transfers is generally significant, because the platform must price several factors into transfers. Every byte transferred:

* consumes bandwidth during transaction propagation

* delays block propagation due to to additional data (because it increases the inter-node transfer time).

* delays block propagation due to additional computation (because the data must be processed when verifying the block at each hop).

* must be stored in the blockchain forever.

* needs to be read from HDD/SSD when requested by a peer

* needs to be stored in RAM when processed.

Each of these factors represent a cost to the network. This proposal presents a technique to transfer data to smart contracts and to network nodes at very low costs, thus enabling use cases that were previously impractical due to gas cost.

# **Specification**

A new type of transaction version 2 is defined. This can be done according to RSKIP145 or another transaction versioning standard. The transaction adds a new field called ephemeralCalldata. However, when transactions is signed, this the data is transformed. Normally, to sign a transaction, the transaction has must be computed. If this proposal is implemented with RSKIPXXX versioning, the calldata field is replaced by the RLP tuple (calldata, ephemeralCallDataSize, ephemeralCalldataRoot) when the transaction hash needs to be computed for signing. The calldata field is renamed **calldataSummary**. In case of RSKIP145 versioning, the fields ephemeralCallDataSize, ephemeralCalldataRoot are added separately to compute the transaction hash. The field calldata is the previously used value and it is unaltered. The field ephemeralCallDataSize represents the size of the ephemeral calldata, and must not contain padding zeros. This encoding enables the removal of the ephemeral data. The field **ephemeralCalldataRoot** is defined as buildMerkleRoot(ephemeralCalldata) The function **buildMerkleRoot** merkelizes the ephemeral calldata and returns the root hash. The merkelization fills all unused intermediate or leaf nodes with zero data. The maximum size of ephemeralCalldata is 100,000 bytes.

An epoch is an interval of blocks M blocks. We propose M=180,000. Depending on the average number of uncles per block, an epoch can a last from 1 month to approximately 2 months.

An ephemeral transaction can use ephemeral calldata. For ephemeral calldata, the first 64 bytes cost 68 gas each. The the remaining bytes costs 8 gas each. 

A transaction can send ephemeral data to the blockchain by the combination of two operations: specifying the field ephemeralCalldata in the transaction, and calling any contract that calls `registerEphemeralCalldata(uint id)` in the precompiled contract Context. If this method is not called, the ephemeral calldata is immediately erased and the block does not need to transfer it. When `registerEphemeralCalldata()`  the Context is called, the ephemeralCalldataRoot, the id passed as argument and the block number and the transaction id are stored inside the Context contract.  Internally, the Context contract uses two dictionaries **epByBlockNumber** and **epBySID**. The acronym SID stands sender and id, and is built as the the keccak hash of those fields concatenated. The dictionary  epByBlockNumber is indexed by block number. Each entry contains another dictionary **epEntryMap**.  The dictionary epEntryMap is indexed by SID and contains the transaction hash, the ephemeralCalldataRoot and the ephemeralCallDataSize. The epBySID  dictionary is indexed by SID and contains the transaction hash, ephemeralCallDataSize,  ephemeralCalldataRoot and block number.

The ephemeralCalldataRoot value can be obtained in the transaction the ephemeral calldata is included by calling the method `getCurrentEphemeralCalldataRoot()` of precompiled contract Context.

Ephemeral calldata can be retrieved by contracts by calling the methods:

*  `getEphemeralCalldataSize(address sender, uint256 id, uint32 blockNumber )` 
* `getEphemeralCalldata(address sender, uint256 id, uint32 offset,uint32 maxSize)`
* `getEphemeralCalldataRoot(address sender, uint256 id)` 

These methods are provided by the precompiled contract Context. If the ephemeral calldata is registered at block at height X, then it will be available to be retrieved in the block range [X+1..X+M+1]. The node can store the ephemeral data longer, but at height greater than X+M+1 it becomes unavailable to smart contracts. If the same data is re-sent in another transaction, and the data is registered under the same id by the same sender, it may last more blocks. In this case the entries in the Context dictionaries will be updated. If the ephemeral calldata was not sent, or not registered, or if it became unavailable, then: 

* `getEphemeralCalldataSize()` returns zero 
* `getEphemeralCalldata()` fills the return buffer with zeros up to maxSize
* `getEphemeralCalldataRoot()`  fills 32 bytes of the return buffer with zeros

The Context smart-contract also has a method used by the node for house-keeping: `getEphemeralDataPersisted()`. When called at block height X, this method returns the transaction hashes of transactions containing registered ephemeral calldata that need to be persisted because they have been queried, and that been included in block (X-2M). This method is not available to be called by onchain transactions, to enable backward compatible upgrades to the Context contract. The method `getEphemeralCalldataPersisted()` does not report persistent transaction hashes for any other block than the one at height (X-2M). How the node removes the ephemeral calldata from transactions is implementation-dependent. An easy method is that the node calls `getEphemeralCalldataPersisted()` at every block at height X, and scans all transactions in block (X-2M) for transactions containing ephemeral calldata, and updates all transactions having ephemeral calldata for transactions not included in the set returned by `getEphemeralCalldataPersisted()`. A more efficient method would be to store the ephemeral calldata separately.

The gas cost of `getEphemeralCalldata()` is 2000 plus an conditional cost C. If the ephemeral calldata was successfully returned before by any contract, C=ephemeralCallDataSize/16. If the ephemeral data is available but it was never retrieved, then C=ephemeralCallDataSize*60. Therefore `getEphemeralCalldata()` maximum gas cost is 620,000.

The method `getEphemeralCalldataRoot()` can be used by smart-contracts to to verify fraud proofs specified with Merkle paths in the ephemeral calldata without retrieving the calldata itself, which reduces the cost of fraud proves, making them affordable to most users. The gas cost of  `getEphemeralCalldataRoot()` is 2000.

The gas cost of `getEphemeralCalldataSize()` is 2000.

Assuming the block has a 8M gas limit, and that 50% of the space is used to store ephemeral data, then  block can contain up to 500 kilobytes of ephemeral data, split into 5 transactions. It this is used to store BLS signatures (32 bytes each), it can pack 15k signatures. If the average block rate is 30 seconds, this implies 500 tps. The remaining block space can be used to store the transaction data for a rollup.



A block having M confirmations is considered settled and cannot be reverted, and it's called a **settled block**.  A block is **immutable** when it has 2M valid confirmations. A node can remove the unreferenced ephemeral transaction calldata from the local storage once the block becomes immutable. 



# Rationale

I’ll introduce Ephemeral Data with an analogy: the Schrödinger’s cat paradox. In this thought experiment related to quantum mechanics, a cat is "locked in a steel chamber, wherein the cat’s life or death depended on the state of a radioactive atom, whether it had decayed and emitted radiation or not. According to Schrödinger, the Copenhagen interpretation implies that the cat remains both alive and dead until the state is observed.". The same idea can be applied to transaction data. The “Schrödinger’s” data can be sent to the blockchain in a new special transaction field that remains invisible to consensus unless a smart-contract refers to it. When the data is observed, it becomes part of the blockchain forever. If the data is not observed, the data disappears and all that remains on-chain forever is the data hash digest (and later we’ll see that even this hash may be disposed).

The cost of adding Ephemeral data to a transaction is related to the bandwidth consumption to transfer it, but the sender does not need to pay (immediately) for eternal blockchain storage.

The ephemeral data is a shorthand for Segregated, Conditionally-Stored and Delayed Data (SCD). We describe what each word means.

**Segregated**

Ephemeral calldata is not directly hashed along with the other transaction data to be signed, only the hash digest of the Ephemeral calldata is. Therefore when Ephemeral calldata is disposed, only the hash digest remains. This pattern is sometimes used for witness data.

**Conditionally-Stored**

Ephemeral calldata only becomes part of the blockchain if it is retrieved by a contract using the query methods. If it is not used before a deadline, the data is removed. Full nodes can be optimized to store ephemeral calldata in RAM, when the ephemeral period is short. This way the full node can reduce the number of storage accesses.

**Delayed**

Ephemeral calldata is delayed, by request of the transaction sender. When delayed, a smart-contract can only access the ephemeral calldata starting from the following block block of ephemeral calldata inclusion. This removes the ephemeral calldata from the critical block propagation path, and allows the ephemeral calldata to be sent after the block, so that transmission occurs in parallel of block validation at the destination node. Even if there is a risk that a miner creating a block does not provide the ephemeral calldata after the block, forcing the destination peer to backtrack, this is the same risk that a node undergoes when processing a block full of transactions in case the last transaction is invalid. 

### Ephemeral Calldata Use Cases

Ephemeral calldata has many use cases. We describe some of them.

**Rollups**

A Rollup is a blockchain scaling technique that moves transaction execution offchain, while keeping the transaction data onchain. The rollup operator performs periodic commitments on the transaction data and ledger state after the transactions were performed. Operators pre-deposit security bonds that are slashed in case they misbehave by presenting transaction data that does not lead to the committed state. Generally users do not need to inspect the transaction data, and they can trust the commitment, as long as there are other systems (generally called watchtowers) that are checking the correctness of the commitments and can alert of fraud though **fraud proofs**. Fraud proofs can only be presented for a period after the operator commits to a state. A rollup transaction has two components: the description and the signature. The transaction description generally must be preserved for users to be able to reconstruct the rollup ledger state in case the operator fails (the exception is if the full ledger is checkpointed in the blockchain). However, after some time, there is no need to store the signatures anymore, because fraud profs for those transactions are not possible. Signatures can represent from 50 to 85% of the transaction data depending on the compression and signature scheme used in practice.

**Fair Multi-player Games**

When the number of participants taking part of a multi party protocol is high, the risk of communication interruptions rises considerably. Imagine a fair multi-party game with the following properties:

- Each participant must act at frequent intervals
- The time for a participant to act is bounded 
- Each action can be described by a binary data string
- Onchain verification of an action is very expensive in terms of gas 
- Each participant verifies that actions made by the remaining participants because the participants do not trust each other


Clearly, if a participant does respond to offchain communications, the remaining participants can challenge the unresponsive participant to post his actions as onchain data, to continue playing. The fairness of the system is increased if the action data is ephemeral calldata because if posting calldata is expensive, challenges can be used as an attacks when the stake at the game is low. This use case is closely related to state channels. A state channel is frequently implemented as a contract that works as an arbiter when a party tries to cheat. But allowing data to be sent as ephemeral calldata is particularly useful when the amount of data and the number of participants is high. The cryptocurrency network can be used as a secure broadcast medium on demand.

**Cheap Secure Push Oracles**

Smart contracts require oracles. Oracles bring information of the outside world into the blockchain  There are two models for using information provided by oracles in a smart contract: the pull model and push model. In the pull model, a user contract queries an oracle contract in a certain transaction in a block. The owner of the oracle (in practice, a computer system obeying the commands of the oracle contract) performs the real-world query and sends a message to the oracle contract containing the information in a following transaction.  The oracle contract then calls-back the user contract, providing the requested information. This model is slow and expensive, as it requires one additional external transaction and an additional internal call, increasing the amount of  gas consumed for each query. This model is generally useful when the information is seldom requested more than once: for instance, a complex call to a server exposed JSON-RPC API. The oracle contract can implement a cache of previously fetched values to lower the average query cost when multiple equal requests occur over a short period of time (i.e. the same block). In the push model, the oracle owner periodically pushes information into the oracle contract. This model is better suited for values that are commonly requested and easily tabulated,  for example, exchange rates. Also the push model allows users to know more precisely which information will be consumed by the contract before making a call to it. A user does not know for sure the actual value a pull oracle will provide before the fact. Oracle creators can use the push model making oracle data ephemeral calldata. Therefore the oracle data is only persisted in the blockchain if it is consumed, and the cost of maintaining the push oracle is greatly reduced.



# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).