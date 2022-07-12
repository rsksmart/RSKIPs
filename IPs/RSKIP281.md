---
rskip: 281
title: Rollup-optimized Ephemeral Calldata
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-10-05
---
# Rollup-optimized Ephemeral Calldata

|RSKIP          |281           |
| :------------ |:-------------|
|**Title**      |Rollup-optimized Ephemeral Calldata|
|**Created**    |5-Oct-2021 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     ||

# **Abstract**

This RSKIP proposes a way to send data to contracts (called ephemeral data) that enables the disposal of this data at a later block, without causing a fork of the blockchain. This RSKIP is an improvement over [RSKIP214](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP214.md) and [RSKIP28](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP28.md), in three ways.  First, it adds the option for the use of contract-defined hash functions for the merkelization of the ephemeral data or for faster ZK proof processing. Second, it is optimized for large calldata payloads, which are used in rollup commits. Third, it provides maximum flexibility enabling both keep-by-default and remove-by-default semantics.


# **Motivation**

A current trend for scaling blockchains, pioneered by Ethereum, is the creation of multiple rollups (called L2s) on top of the onchain layer (called L1). While there is an ongoing debate over if a single onchain layer (L1) should scale to support billion of user payments or this should be delegated to L2s such as rollups, rollups provide an scaling testbed that can be explored now, without incurring in high risks of modifying the L1. Rollups use mostly the L1 as a data availability layer, creating periodic commit transactions containing compressed transaction data that must persist while the L2 is active. The committed data enables onchain arbitration in case of disputes and also it enables users to reconstruct the L2 blockchain state in case the operators disappear or censor transactions. However, this transaction data could also be disposed if an L2 performed periodic full state commitments (or snapshots) on the L1. Even better, the snapshot could be compressed by specifying only the differences with the previous snapshot, what we call *state diffs*. 

While we don't have accurate models of how L2 will be used in the coming years, most models of L2 usage assumes there will be a high transaction volume destined for simple token payments. If this is true, then fully describing state changes using the executed transactions requires much more space than fully describing the state using a state diff.  Simple payments have explicit source/destination/amount/token data and if two payments modify the same address balance (either source or destination), then at least one address and one balance field would be saved if the state change would be specified in terms of state data diff instead of in terms of a series of transactions. All transitive payments are compressed when performing a state diff. 

We'll define some terms to characterize transactions. We call *data-dominated* transactions to transactions whose output state changes can be specified at a lower gas cost by giving the changes explicitly in textual form, than executing the transaction and collecting the resulting state changes . We'll call *CPU-dominated* to the others. Simple payments are clearly data-dominated. A CPU-bound transaction would be one that performs many changes to the state programmatically, without any explicit key/value information indicated in the transaction calldata. For example, a transaction that executes a contract that sends 1 satoshi to 10000 sequential account addresses starting from a fixed address. 
While we could use the CPU-bound and storage-bound definitions from [RSKIP62](
https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP62.md), we think the definitions proposed here are easier to reason about. While we can't assure that all L2s will be data-dominated, it is highly probably that at least certain L2s will focus on cheap token payments and will be data-dominated.



## Savings in Data-dominated L2s

A data-dominated L2s could expect a considerable reduction in transaction costs if the operator(s) could:

- post periodic L2 blocks containing transactions at an average interval tT (i.e. tT=1 minute)
- post periodic L2 state diffs at a different average interval tD (ie.e tD = 1 month) creating state checkpoints.
- Remove all L2 blocks previously posted to the blockchain between two old state checkpoints, when the probability that the blockchain is reorganized to remove the checkpoints involved is negligible.

This idea was analyzed in the past under the name "ephemeral calldata" in proposals [RSKIP28](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP28.md) and [RSKIP214](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP214.md). In this RSKIP, we use the acronym ECD for ephemeral calldata for brevity.

The main challenge of designing an efficient ECD scheme is making the cost of removing the old data from the blockchain much lower than the cost of the data overhead itself. A secondary challenge is designing and implementing a more complex blockchain synchronization protocol that is required to broadcast blocks with incomplete data.

## Keep-by-Default vs Remove-by-Default

There are two types of ECD: keep-by-default (KBD-ECD) and remove-by-default (RBD-ECD).  RSKIP214 proposes RBD-ECD. RBD-ECD has the benefit that the user posting the RBD-ECD pre-pays only for the cost of temporal storage plus the the cost of data removal, which must be lower than the cost of data persistence. In case ECD needs to be persisted forever, the user or contract that needs to persist the data pays for this cost. This avoids inter-block refunds, which in turn avoids temporary storage of gas that can lead to gastokens. However, RBD-ECD puts L2 at a higher risk that any malfunction or censorship of the transactions that commands the ECD to persist will lead to the data becoming unavailable. On the contrary, KBD-ECD protects L2 users by default.  Last, KBD-ECD can emulate RBD-ECD by adding some user contract method that enables anyone to trigger the ECD removal functionality by calling the method, and paying a small bounty for the user that performs this call.

Because we think the two semantics are useful, this proposal enables both KBD-ECD and RBD-ECD.

# **Specification**

We split the specification in different sections covering transaction format, execution, removal workflow , querying methods and blockchain synchronization.

## Transaction Format Extension

A new type of transaction version 2 is defined. This can be done according to [RSKIP145](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP145.md), [RSKIP212](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP212.md), [RSKIP213](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP213.md) or another transaction versioning standard chosen by the community, such as Ethereum's [EIP2718](https://eips.ethereum.org/EIPS/eip-2718). 

Let U be the tuple (**ecdSize**, **hasherContractAddress**, **hasherCallGasConsumed**, **ecdStdHash**, **ecdHash**,**keepByDefault**). If this proposal is implemented with RSKIP212 versioning, the calldata field is replaced by the RLP encoding of the RLP tuple (**calldata**,  **U** ). The calldata field is renamed **calldataSummary**. The double RLP encoding ensures  that the outer most encoding is still a single element and not a list, preserving hardware wallet compatibility. 
If this proposal is implemented with RSKIP213 versioning, then a new field **ecdInfo** can be added to the transaction containing the tuple U or an empty value (if no ECD is present). 
In case of RSKIP145 versioning, the fields in the tuple U are added separately, each one with a new field id. 

In case of EIP2718 versioning, a new transaction type is used and arguments can be added at the end of the existing transaction fields.

Here we explain each the new fields that wallets must handle. Some of these fields are **internal**, which means that they are computed from some other fields. 

- **ecdSize**: size of ecd.
- **ecd**: the calldata that can potentially be removed
- **ecdStdHash**: the Keccak hash of the ecd. This hash is only checked if the ECD is present.
- **ecdHash**: the hash digest of the hasher specified by hasherContractAddress, applied to the **ecd**.
- **hasherContractAddress**: the contract address of a hashing function. The value zero indicates no hasher contract will be used.
- **hasherCallGasConsumed**: the gas consumed when hasherContractAddress is called. Must be zero if the hasherContractAddress is zero.
- **keepByDefault**: if true (1) the ECD is kept by default, If false (0), it is removed by default.

Only the **ecd** and **hasherContractAddress** fields are chosen freely by the user's wallet, the other fields are derived from **ecd** and are computed by the wallet.

The value **ecdSize** must match the size of the **ecd**, if not, the transaction is invalid.

When a transaction is serialized for wire transmission, the transaction is wrapped in a RLP object containing two values. The first is the unwrapped transaction, and the second is the **ecd**, which can be null if absent. 

It is highly probable that nodes will store the **ecd** in a separate database, indexed by block hash and transaction index, to ease its removal.

## Computation of  ecdHash

The value **ecdHash** is computed by the wallet as `hasherCall(ecd)` where `hasherCall()` represents the off-chain call (using `STATICCALL`) to the **hasherContract** that resides at the **hasherContractAddress**. The call takes **ecd** as argument. The `hasherCall()` is given **ecdSize**\*10 gas for execution. If it ends with `OOG` or if it executes`REVERT`, then the transaction is invalid. The gas consumed by this call must match exactly the value of the field **hasherCallGasConsumed**.

The **hasherContract** must be have been created with `CREATE` opcode (or transaction with null destination). It cannot be a contract with changeable code (such as contracts created with `CREATE2` opcode). To achieve this, contracts store a flag indicating how they were created: this is specified a separate RSKIP to be defined.

When called, `hasherCall` must return exactly 32 bytes. The `hasherCall` cannot read or store data on its storage, nor perform calls to other contracts with opcodes other than `STATICALL`. If `hasherCall` reverts, throws `OOG` or returns anything else, the transaction becomes invalid.

Nodes must perform the corresponding `hasherCall` when they receive a mempool transaction with **ecd**, and verify the correct signature before broadcasting it again. Note that depending on how the **hasherContract** contract is 

The minimum size for **ecd** is 1024 bytes. If less data is specified, the transaction is processed as normal, but the **ecd** cannot be removed from the blockchain later.

The initial cost of **ecd** is 8 gas per byte, eight times lower than the cost of calldata. This accounts for the cost of transaction data propagation. 
The destination address of the transaction containing the ephemeral data becomes the ephemeral data owner (**ecdOwner**). 

### Avoidance of ECD Malleability

Since the computation of the hash that identifies the ECD is specified by the user, it's possible that the user chooses a hasher contract implementing a flawed hash function (i.e. non-cryptographic or simply a constant function). To avoid malleability, if the ECD of a transaction is included in a block, then the **ecdStdHash** must match the `Keccak` hash of the **ecd**.



## ECD Registration

A new precompiled contract `EphemeralDataManager` exists at address `0x00..0100001B`. The contract has several new methods related to handling ephemeral calldata.

We define the SID as the `KECCAK` hash of the fixed 64-byte message of the sender and id fields concatenated. ("S" in SID stands for Sender).

The **ecdOwner** can register the ephemeral data by calling the method: 
`registerEphemeralCalldata(uint id) returns(uint32 err)` 

The registration occurs only if this call does not revert. The id value can be chosen by the caller in any way.  As examples, it can be a sequential number incremented on every call or it can be the concatenation of the block and transaction number. the The, together with the sender, form the SID. 

The method succeeds if:

* The transaction has ECD.

* The sender is the **ecdOwner**.

* The ECD has not been already registered successfully.

* The SID does not already exists in the `epBySID` map.

* The **depositAmount** is transferred with the call.

  

On success:

* A new entry in the the `epBySID` map is created.

* An amount of 10K gas will be consumed (accounts for the storage of the entry).

* If **keepByDefault**==false, then an additional amount of 20K gas will be consumed (accounts for later removal)

* logs an event with the following signature:

  `ecdRegistered(uint256 sid)`

* returns 0.

  

On failure:

* An amount of 800 gas is consumed
* Returns 1 (if the transaction does not have ECD) , 2 (the sender is not the **ecdOwner**), 3 (if the ECD was previously registered), 4 (if the `epBySID` entry exists), 5 (if the **depositAmount** is not transferred).

 The method will succeed only once. If called more than once (with the same or a different id), it will fail with error code 3.

If (**keepByDefault**==true), then a bitcoin amount **depositAmount** must be transferred to the `EphemeralDataManager` contract with the `registerEphemeralCalldata()` call. The **depositAmount** is defined gasPrice\*`getGasDeposit()`.  The auxiliary function `getGasDeposit()` is defined as (**hasherCallGasConsumed**+48***ecdSize**). 

This amount is locked in the contract, and it is only reimbursed to the user if the user later removes the ephemeral data by calling `removeEphemeralData()`.

If there is ECD in the transaction and the method `registerEphemeralCalldata()` is not called, the ECD is immediately erased and the block containing the transaction does not need to transfer the ECD. 

When `registerEphemeralCalldata()` succeeds, the following data is stored inside the map `epBySID` in the `EphemeralDataManager` contract:

* uint256 ecdHash
* uint256 id passed as argument
* uint32 blockNumber (the current block number)
* uint32 transactionIndex (the current transaction index in the block) 
* address ecdOwner (also known as caller or sender)
* uint256 amountDeposited (amount in bitcoins deposited)
* uint32 hasherCallGasConsumed
* uint32 removalRequestBlockNumber
* uint32 removalRequestTransactionIndex
* uint8 state (the current state of the ECD)

The entry size is 137 bytes and it can be stored in a single Unitrie storage cell to reduce storage costs.

The dictionary `epBySID` dictionary is indexed by SID.  Each entry contains the other fields. The state of a new entry is *removable* if keepByDefault is true, or *autoRemovable* if false. 

The fields **removalRequestBlockNumber** and **removalRequestTransactionIndex** will start as zero, and will be filled when the removal request is issued.

The **state** can be one of four possibilities:

- 0: *unknown* (indicating the entry does not exists)
- 1: *autoRemovable* (indicating the ECD will be removed automatically if not queried)
- 2: *removable* (indicating the ECD is registered and can be removed if not too late)
- 3: *toBeRemoved* (indicating the a request to remove the ECD was received but it hasn't been removed yet)
- 4: *permanent* (indicating the ECD will cannot be removed anymore)

The ECD can be removed only if the state is *removable* and the deadline to remove it has not elapsed.



## ECD Removal Request

A contract can start the removal process of an ECD by calling a method `requestRemovalOfEphemeralData()` that exists in a precompiled contract `EphemeralDataManager` with the following ABI:

`function requestRemovalOfEphemeralData(uint32 id) returns (uint32 error);`

The contract method will build the SID based on the sender address and the id passed as argument.

The method succeeds if: 

- There is an entry in `epBySID` for the SID key built with the sender and the id.
- The state of the entry is *removable*.
- The current block height is less than the block height where the SID was registered plus a constant M. 

On success: 

- The `epBySID` entry is updated: the state is set to *toBeRemoved*. 

- The fields **removalRequestBlockNumber** and **removalRequestTransactionIndex** are filled with the current block number and transaction index. This data will be used when the actual removal occurs.

- An amount of gas equivalent to 10000 is consumed (this accounts for the storage changes).

- logs an event with the following signature:

  `ecdRemovalRequested(uint256 sid,uint32 blockNumber,uint32 transactionIndex)`
  where both arguments correspond to the block number and transaction index when the ECD was registered.

- `requestRemovalOfEphemeralData()` returns 0

The value M is defined as 180,000. Depending on the average number of uncles per block, an epoch can a last from 1 month to approximately 2 months.

If data of the **ecd** is requested by the query method (see later), the state will be *permanent* and then the data cannot be removed.

On failure:

* The gas consumed is 800
* The method returns 1 (if the entry was not found) or 2 (if the state was not *removable*) or 3 (if the block height was not within the required bounds)



## ECD Removal

Once a delay D=30000 blocks have elapsed since the ECD entry has been requested for removal, then the method `removeEphemeralData()` can be called. The method signature is the following:

`function removeEphemeralData(uint32 id) returns (uint32 error)`

The delay, which  corresponds to approximately 10 days, prevents blockchain reorganizations to cancel removal of ECDs. The method `removeEphemeralData()` actually removes the ECD from the blockchain. the SID is computed as before.

The method succeeds if: 

* the entry specified by the SID exists 
* the state is *removable*. 

On success:

- a gas of 30000, is consumed ( which accounts for the storage changes (10K) and changing the transaction record of an old block (20K)). 

- logs an event with the following signature:

  `ecdRemoved(uint256 sid, uint32 removalRequestBlockNumber,uint32 removalRequestTransactionIndex)`. Here the arguments correspond to the block number and transaction index when the ECD removal was requested (`ecdRemovalRequested` event).

- Transfers back to the caller the deposited BTCs (**amountDeposited** value). This transfer is made as a direct balance increase, not as a recursive call.

- returns 0

On failure:

*  the gas consumed is 800.
* returns 1 (if the entry was not found) or 2 (if the state was not *removable*)

## ECD Persistence Request

A contract persist an ECD by two methods: querying data from the ECD to be used in a contract or requesting the persistence of the data in the blockchain without actually using the data. Here we explain this second option. To persist the ecd, a contract can call the method `requestPersistenceOfEphemeralData()` that exists in a precompiled contract `EphemeralDataManager` with the following ABI:

`function requestPersistenceOfEphemeralData(uint32 id) returns (uint32 error);`

The SID is computed as before.

The method succeeds if: 

- There is an entry in  `epBySID` with the SID key.
- The state of the entry is *autoRemovable*.
- The current block height is less than the block height where the SID was registered plus a constant M. 

On success:

* sets the state of the ECD entry to *permanent*.

* consumes the gas computed by `getGasDeposit()` (this accounts for the fact that the nodes will need to call the hasher each time the transaction is verified)

* consumes 5K gas for the storage change (since 30K was pre-paid for the removal and removal was not performed, the storage change is subsidized by 5K)

* Logs the event with the following signature:

  `ecdPersisted(uint256 sid)`.

* returns 0

  

On failure it consumes only 800 gas, and returns 1 (if the entry was not found), 2 (if the state was not autoRemovable) or 3 (if the block height is higher than the required bound).



## Ephemeral Calldata Retrieval

Information regarding the ECD specified in the current transaction can be accessed with the following methods: 

* `getCurrentEcdHash() returns (uint256 userHash)` 

* `getCurrentEcdStdHash() returns (uint256 stdHash)` 

* `getCurrentEcdSize() returns (uint32 size)`

* `getCurrentHasherCallGasConsumed() returns (uint32 size)`

* `getCurrentHasherContractAddress() returns (address addr)`

* `getCurrentDepositAmount() returns(uint256 amount)`

* `getCurrentTransactionIndex() returns(uint32 index)`

* `getCurrentBlockNumber() returns(uint32 number)`

More information about certain past ephemeral calldata can be retrieved by contracts by calling the methods:

* `getEcdSize(uint256 sid) returns (uint32 size)`
* `getEcd(uint256 sid, uint32 offset,uint32 maxSize) returns(bytes data)`
* `getEcdHash(uint256 sid) returns (uint256 userHash)`
* `getDepositAmount(uint256 sid) returns(uint256 amount)`
* `getHasherContractAddress(uint256 sid) returns (address addr)`
* `getTransactionIndex(address sender, uint256 id) returns(uint32 index)`
* `getBlockNumber(uint256 sid) returns(uint32 number)`
* `getEcdState(uint256 sid) returns(uint8 number)`


These methods are provided by the precompiled contract `EphemeralDataManager`. 

Once `getEcd()` is called, the **ecd** cannot be removed anymore, and attempting to removed it returns an error code. Internally, the when is called the  `epBySID` map entry is marked by setting the state to  *permanent*. The initial gas cost of `GetEcd()` is 5000. If the state of the ECD was *autoRemovable*, then the gas computed by `getGasDeposit()` is also consumed.

`GetEcd()` will fail if called specifying a SID successfully registered in the same block.


The gas cost of all the remaining query methods (i.e. `GetEcdSize()` and `GetEcdHash()`) is 1000 each.

If the ECD is registered at block at height X, then the entry data will be available to be retrieved in the block range [X+1..X+M+1]. The node can store the ephemeral data longer, but at height greater than (X+M+1) it becomes unavailable to smart contracts. If the ECD was not sent, or not registered, or if it became unavailable, then:

* `GetEcdSize()` returns zero
* `GetEcd()` fills the return buffer with zeros up to maxSize
* `GetEcdHash()` fills 32 bytes of the return buffer with zeros
* `getDepositAmount()` returns zero
* `getHasherContractAddress()` return zero
* `getTransactionIndex()` return zero
* `getBlockNumber()` returns zero

# Blockchain Synchronization

A transaction with an non-zero ecdSize is called *mutable*. A block that has at least one mutable transaction is called *mutable block*.  If it has no mutable transactions it is *immutable*. If a block is mutable and has all its ephemeral data or if it is immutable, it is called *complete*. We *ECD inclusion rule* to the consensus rule that governs if ephemeral data must be present or not in a block for the block to be accepted. The rule will be defined shortly. Any definition on a block that affected by the ECD inclusion rule will make the block state depend on future blocks, and therefore this state could change. We call *core consensus rules* to the set all consensus rules except the ECD inclusion rule.  Core consensus rules do not depend on future blocks. A block that with mutable transactions that breaks the ECD inclusion rule is called *deficient*. A block that satisfies the core consensus rules is called *adequate*. A block that is adequate and respects the ECD inclusion rule is called *valid*. If not, it is *invalid*. Note that the validity of a  block could change if future blocks are reorganized. A valid block that has (M+D) valid block confirmations is called *sealed*. 

To synchronize the blockchain, nodes keep three block pointers that specify block heights. The *tipBlock* , *shrinkableBlock* and *sealedBlock*.  All blocks with height equal or lower than the sealedBlock are sealed. All blocks with height equal or lower than shrinkableBlock are valid, but could have their ephemeral data removed (but would still be valid).

The pointers satisfy sealedBlock <= shrinkableBlock <= tipBlock. The tipBlock points to the last adequate block received. All blocks in the local blockchain are adequate.

Let the block B be the child of the shrinkableBlock. The state of B can change according to the following algorithm: 

* if the next block (height i) is complete, then the block B is sealed and shrinkableBlock advances.

* if all blocks j (i<j<=i+M) exists in the blockchain and they are either sealed or complete:
  * For each transaction T in the block i that had ECD but it was removed:
    * if an event `ecdRegistered(sid)` is not present in its receipt, skip this transaction.
    * if (T.keepByDefault==true), then 
      * if any block j (i<j<=i+M) do not contain the event `ecdRemovalRequested(uint256 sid,..)` generated by the `EphemeralDataManager` contract , then the block is deficient.
    * if (T.keepByDefault==false), 
      * if any block j (i<j<=i+M) contain the the event `ecdPersisted(uint256 sid)`generated by the `EphemeralDataManager` contract, then then the block is deficient.
  * If the block is not deficient, then shrinkableBlock can advance.

The algorithm that governs the sealedBlock pointer is the following:

* If (i+M <= shrinkableBlock) then increment sealedBlock 
* if  (i+M <= tipBlock)  and blocks (i..i+M] are complete, then increment sealedBlock 

The algorithm that governs the tipBlock pointer is the following:

* When a new block arrives, if it is adequate, add it to the blockchain and make tipBlock point to it.
* If the new block emits an event `ecdRemoved(sid, removalRequestBlockNumber,removalRequestTransactionIndex)`, then query the contract to find the corresponding mutable transaction T (by block number and transaction index). Then remove the ECD of this transaction from the blockchain. The height of the block containing T should be higher than the sealedBlock pointer but lower or equal to the shrinkableBlock pointer.  

When a node is fully synchronized, the latest M blocks of the blockchain will be complete, and the tipBlock and shrinkableBlock pointers will match and point to the same block. The pointer sealedBlock will point to a block M blocks behind.

If an incomplete block is detected during synchronization, then it is necessary either reorganize the chain or look for the missing data from a another source peer. The node can first try to find the missing data, and if that's not possible, find a peer having an alternate fork. However, to avoid selecting the wrong fork because of transient network problems, it is recommended that nodes only switch to a new fork if this fork has higher cumulative work. While no better fork is found, the node should pause, try connecting with new  peers. and wait.



# Rationale

We discuss several design topics in the following sections. The ephemeral data was designed to achieve three properties: Segregated, Conditionally-stored and Delayed.

### **Segregation**

Ephemeral calldata is not directly hashed along with the other transaction data to be signed, only the user-defined hash digest of the Ephemeral calldata is. Therefore  ephemeral calldata can be disposed, and only the hash digest,  the data size, and the hashing cost remains in the blockchain. This pattern is sometimes used for witness data in blockchains.

### **Conditional Storage**

Ephemeral calldata only becomes part of the blockchain if it is retrieved by a contract using the query methods. Contracts can If it is not used before a deadline, the data is removed from the blockchain. Full nodes can be optimized to store ephemeral calldata in efficient databases that enable removal without re-packing.

### **Access Delay**

Ephemeral calldata cannot be accessed in the same block it is transmitted, but it is delayed one block. A smart-contract can only access the ephemeral calldata starting from the following block of ephemeral calldata inclusion. This reduces the stress to process the ephemeral data in the critical block propagation path from miner to miner. The ephemeral calldata can be sent after the propagation of the standard block data, so that ephemeral data transmission occurs in parallel of block validation at the destination node. A miner can compute the resulting state of a received block before the ECD has been received. Even if there is a risk that a miner creating a block does not provide the ephemeral calldata after the block, forcing the destination peer to backtrack, this is the same risk that a node undergoes when processing a block full of transactions in case the last transaction is invalid. 

### User-defined Hash Function

The goal of enabling a user-specified hash function is two-fold:

- If the hasher computes a Merkle-tree of the ECD and returns its root, it enables selective disputes that require proving that a certain element in the **ecd**. 
- If the ecd is used as L2 transaction data for a a ZK-rollup, then the rollup designer may choose to implement a SNARK-friendly hashing function, so that the ZK proofs can be created with much lower resources.



# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 

# Test Cases

TBD

## Security Considerations

No new practical denial of service or resource abuse risks have been identified related to this change. However, we identify a number of potential theoretical problems.

### Weird Blocks

If an ECD is removed, it's possible that the **ecdSize** and **ecdHash** contain invalid values, that never matched the **ecdStdHash**. Let define a transaction T to be *weird* if it can switch from invalid to valid state after M confirmations when the ECD is removed. Weird transactions violate some a common assumption in blockchain protocols, but it does not seem to cause problems in practice. Let define a *weird block* as a block containing weird transactions. Since weird block would be flagged as invalid (and this flag may persist) it is possible for an attacker to attempt to split the network by creating a weird block with M confirmations. First, the weird block alone is sent to half of the miners, and they reject it. Then the weird block but without the ECDs, along with M confirmations is sent to the other half. This other half will accept the fork. This attack is impractical since creating M consecutive blocks for a minority miner is infeasible.

 


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
