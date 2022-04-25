---
rskip: 62
title: Compressed block propagation using state trie update batch
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2018-05-07
---
# Compressed block propagation using state trie update batch (COBLO)

|RSKIP          |62           |
| :------------ |:-------------|
|**Title**      |Compressed block propagation using state trie update batch (COBLO) |
|**Created**    |07-MAY-2018 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

Stateful turing complete blockchains poses a limitation that is not present in UTXO based blockchains: one transaction state can affect the following transaction pre-state, and therefore transactions cannot be pre-executed during the transaction propagation phase. Also the block  contact (e.g. timestamp, number, coinbase) can also affect how a transaction is executed. Therefore transaction execution is directly obstructing the critical path during block propagation. The longer the times it takes for a block to propagate, the higher the uncle rate, the lower the blockchain efficiency and fairness. There are two kinds to solutions to this problem: even/odd sharding and headers-first propagation. Even-odd sharing consist of splitting the blockchain state into two, one state that is updated on even blocks and another that is updated on odd blocks. Therefore miners can mine child blocks even if the parent block has not been executed. The drawback is that a queue-based communication mechanism between shards must be devised.  The standard headers-first solution does not work in RSK/Ethereum because the state trie needs to be updated with the parent block execution results, even if an empty block is mined. In this RSK we present a modification of this headers-first technique that allow miners to start mining a child block before the parent block has been fully verified, based on forwarding the batch of state modifications along the header of the block.

## Discussion 

There are two kinds to solutions to this problem: even/odd sharding and headers-first propagation. Even-odd sharing consist of splitting the blockchain state into two, one state that is updated on even blocks and another that is updated on odd blocks. Therefore miners can mine child blocks even if the parent block has not been executed. The drawback is that a queue-based communication mechanism between shards must be devised.  The standard headers-first solution does not work in RSK/Ethereum because the state trie needs to be updated with the parent block execution results, even if an empty block is mined. In this RSK we present a modification of this headers-first technique that allow miners to start mining a child block before the parent block has been fully verified, based on forwarding the batch of state modifications along the header of the block.

# **Specification**

A compressed block (COBLO) format is defined. The new object has the following fields:

1. blockHeader: A block header (including uncles’s headers and Bitcoin SPV proof)
2. compressedTransactionIds
3. changeBatch (see below)



The compressesTransactionIds is a list of elements of type CompressedTransactionId. A CompressedTransactionId is a 10-byte prefix (80 bits) of a 256-bit transaction id.

The changeBatch field is used to specify the changes that transactions apply to the world state. The changeBatch field is a list of elements of type Change. A Change identifies the list of changes a transaction performs on the state trie. A Change is a tuple (transactionIndex, transactionChanges). The transactionIndex is the index of the transaction in the block that is being described. If a transaction index is not present in the changeBatch list, then that transaction must be executed in order to detect the changes.

This allows the miner to compress the block by characterizing transactions in CPU-bound and Storage-bound.  CPU-bound transactions are transactions that consume a higher amount of gas in computation than in modifying the storage. Storage-bound transactions are the opposite. The exact threshold is chosen by the miner. The miner will add to the changeBatch all CPU-bound transactions.  

The transactionChanges is a list of elements of type TransactionChange. TransactionChange is a tuple (account,internalChanges,storageChanges). The account is the address of the account (or contract) that this transaction is changing (or being created).  internalChanges is a list of elements of type InternalChange. An InternalChange is a tuple (internalKey,value) where internalKey has a the following meaning:

| internalKey | Meaning                                         |
| ----------- | ----------------------------------------------- |
| 0           | Nonce                                           |
| 1           | Balance                                         |
| 2           | Code                                            |
| 3           | lastChangeDate (for storage rent). See RSKIP61. |
| other       | reserved for future improvements                |

storageChanges is a list of elements of type StorageChange. a StorageChange is a tuple (key,value) where key is the address of a cell in contract storage, and value represents the value to be stored. If the value is empty, then the cell has been removed. Keys are subject to the key-compression scheme defined below



## Key/Value Compression Scheme

 

All keys and values can be compressed (internalKey is excluded). The compression algorithm is a variant of run-length encoding.If first byte is zero, a normal data is read, starting from the second byte. 

If first byte is non-zero, then the three most significant bits are special, and they represent different compression options. These three bits are masked-out when values are extracted. 

Most significant bits table:

| Bits | Meaning                                                      |
| :--- | ------------------------------------------------------------ |
| 000  | the blob is interpreted as (dummyByte,valueData). The resulting data is given by the valueData field. |
| 001  | the blob is interpreted as (dataLength, dataOffset,), each a 16-bit unsigned word (totaling 4 bytes). The value is taken from the “data” field of the transaction at the specified offset and length. If the provided values are invalid, then the block header is invalid. This type of compression is useful to specify that a certain contract will store the code that is specified by a contract creation transaction. |
| 010  | the blob is interpreted as (dummyByte, account, storage-address, valueOffset, valueLength) where account is 20 bytes in length, storage-address is 32-byte in length, and valueOffset/valueLength are 16-bit unsigned. This extracts from the state of a contract a part of the storage. Because RSK allows storage values to be larger than 32 bytes, this compression method allows to extract any number of bytes at any offset. This can be used when a contract creates another contract, extracting the code from storage. This will only work for a new CREATE3 opcode that allows to specify where the code is taken directly by specifying the storage address, offset and length, eliminating the need to “search” for a match. |
| 011  | the blob is interpreted as (dummyByte) and the value copied is the receiver's address of the transaction. |
| 100  | the blog is interpreted as (dataByte) and the resulting data is dataByte (with the most-significant bit cleared). This allows encoding of values from 0 to 31. |
| 1xx  | For xx != 00, is reserved for future use. Block headers should be discarded if they contain such encodings. |

It's important to note that if the node that receives the compressed header does not have a copy of a transaction specified by a compressedTransactionId, it doesn't need to immediately ask for it to a peer. It can forward the compressed block 

## Propagation of COBLOs

A new network message used to propagate COBLOs is created. When a non-miner full node receives a COBLO, it performs the following actions:

1. Connects it to its parent. If the parent is missing, the COBLO is discarded.
2. Verifies its PoW against the difficulty. If the PoW is invalid, the COBLO is discarded.
3. Verifies all block header fields that doesn't need transaction execution.
4. It checks if all the transactions specified in the compressedTransactionIds are in the memory pool. If not, it fetches the missing transactions from the peer that sent the COBLO.
5. Reconstruct the transaction trie from the compressed ids and compare it with the block header trie root.
6. If the transaction does not verify, the node fetches from the sending peer the list of transactions ids of the COBLO (the full ids, not the compressed ids). Then the transaction trie is re-built. 
   1. 1. If the new trie is still different from the trie root specified in the block header, the block is discarded (the sending peer may be banned)
      2. If the new trie matches the root, then it means that there is a transaction id collision. In that case it replaces the compressed id with a full id or the compressedTransactionIds field, and keeps processing the COBLO as normal.
7. If any transaction is missing, the peer ask the sending node to provide it. If the transaction is not provided, the processing of the COBLO is aborted.
8. The node forwards the COBLO to its peers (if some transaction ids are expanded, then the COBLO forwarded will be the one with the expanded ids)
9. The COBLO is executed normally from start to finish, and this time the trie state is verified against all values derived from execution, and not from the updateBatch. If this execution leads to the result that the COBLO is invalid, or if some transactions in the COBLO could not be fetched from peers, the COBLO is removed from the blockchain.



## How Miners process COBLOs

When a miner receives a COBLO, after step 4 and before step 5, the miner adds it to its local best branch by performing the changes specified by the changeBatch and executing the transactions not specified in the changeBatch. In case this COBLO is the new tip of the best chain, it starts mining a child of the COBLO. Even if the transactions in the COBLO are unknown, the miner can build a child block whose transactions ids do not match any of the prefixes specified in the compressedTransactionIds list. 

While mining the child block, the steps following 4 continue to be executed. If this execution leads to the result that the COBLO is invalid, or if some transactions in the COBLO could not be fetched from peers, the COBLO is removed from the blockchain and mining is stopped and restarted from the previous state.

## Attacks

Because the transaction ids are compressed, an attacker may try to create two transaction with colliding compressed ids. The attacker can then send one of them to the miners (or mine it privately) while distributing the other, in order to prevent a COBLO from being forwarded. The attack is detected when the node cannot reconstruct the transaction trie, and the correct transaction will be requested from the sending peer. Afterward a disambiguating COBLO will be broadcasted. Therefore the attack can only delay the propagation of the COBLO a single hop. 

An attacker may try to mine a block and forward its COBLO without disclosing some of the transactions. In this case, in the first hop a peer will ask for the missing transactions and will not keep forwarding it, nullifying any flooding attack.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).