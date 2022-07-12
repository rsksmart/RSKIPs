---
rskip: 1
title: Distributed Memory
description: This RSK contract memory is centralized. For contracts that are infrequently used this is not a problem. 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2009-06-16
---

# Distributed Memory

|RSKIP          |1           |
| :------------ |:-------------|
|**Title**      |Distributed Memory |
|**Created**    |09-JUN-16 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

RSKIP describes a new persistent memory system where data is distributed in user accounts instead of being centralized in the contract. This RSKIP also proposes modifications in the VM and consensus rules to allow scaling by transaction partitioning. The main motivation is preventing bottlenecks in transaction processing.

# **Motivation**

This RSK contract memory is centralized. For contracts that are infrequently used this is not a problem. However contracts that are frequently used impose a bottleneck in transaction processing. If many transactions in a block use that contract, the transactions cannot be processed independently: they must be serialized. The prevents transaction execution to be parallelized in several processing cores. We can assume that the most used contracts will be liquid assets, such as other crypto-tokens and representations of fiat currencies, because these contracts serve as building blocks for dapps, are will be consumed by wallets. As an example, all accounts and contracts that hold or transact in cryptoUSD must communicate with the cryptoUSD issuer contract whenever they need to transfer cryptoUSD.

Most assets will be bearer-instruments, and therefore the issuing smart-contract would not prevent free transactions of the assets. Therefore the issuing smart-contract does not need to access any central memory (at least not write access), and can execute the code accessing the distributed memory of the source and destination accounts (or contracts). IF the token managing contract does not write to local persistent memory, then it can be safely executed in parallel.

## Discussion

Each Account or Contract is added an additional Trie data structure foreignStorage. The keys are the account addresses of the foreign contracts that are holding memory locally. Each cell holds a second level trie. Each key of this second level trie is an arbitrary key provided by the contract. The data is arbitrary (and of arbitrary length). 

Is is also desirable that contracts can inspect their own foreign storage: this can be used by wallet contracts to discover owned assets. However, not all pieces of data stored in the foreignMemory may be assets, so this discovery method would not always work.

To prevent contracts for storing undesired information in a user wallet, contracts and wallets need to authorize such storage. This could be done by new opcodes or by network messages. As accounts do not have code, therefore they cannot hold foreign memory without special network messages. Therefore this RSKIP should be based on RSKIP02, and only allow foreignStorage for smart contracts.

Five new opcodes are added to move bytes from local to foreign distributed storage, and vice-versa.

# **Specification**

### FSSTOREBYTES

Foreign Storage Store bytes.

Arguments: <src_offset> <count> <dst_address> <dst_key> 

Opcode: TBD

Stores in the foreign memory of the contract or account identified by dst_address, at key dst_key,  an array of bytes taken from the memory cell src_offset. If the destination key is already holding information, it is removed and replaced by the new information. If count is empty, the key is removed. If after the deletion of a key there are no more key holding information in the foreign trie, the whole trie is removed.

Gas cost: the basic gas cost is 700. The additional costs are similar to SSTORE. The CLEAR, SET and RESET costs are applied individually for each 32-byte block of data, rounding up count to a 32-byte multiple.

### FSLOADBYTES

Foreign Storage Load bytes.

Arguments: <src_address> <src_key> <src_offset> <count> <dst_offset>  

Gas Cost: 1000

Opcode: TBD

Loads from the foreign contract or account identified by src_address at key src_key an array of count bytes starting from the offset src_offeset and puts it in the byte-addressed memory at offset dst_offset. If access is out range (the foreign item has less than count bytes), a an OOG exception is raised. To obtain the amount of data stored, use the FSSIZE opcode. The basic gas cost is 700. Additional costs of 100 gas per 32-byte chunk accessed, where count is rounded up to a 32-byte chunk.

### FSSIZE

Foreign Storage Size

Arguments: <src_address> <src_key> 

Gas Cost: 1000

Returns the size of the data stored. 

### FSENABLE

Foreign Storage Enable

Arguments: <address>

GasCost: 20000

Opcode: TBD

Disables foreign storage for contract at address specified. 

### FSDISABLE

Foreign Storage Disable

Arguments: <address>

GasCost: 5000

Opcode: TBD

Enables foreign storage for contract at address specified.

### FSENABLED

Foreign Storage Enabled

Arguments: <address>

GasCost: 200

Opcode: TBD

Pushes 1 if contract at address is enabled to use the foreign storage or 0 otherwise.

### New Account State

A new trieRoot field named foreignStorage is added as the last element of the account state. Initially this is an empty trie. The foreignStorage keys are the extenal contract addresses that can store local information. If the key is present, then foreign storage is enabled. The payload for each key is a trie containing (key,value) pairs were key is any 256-bit value and value is any byte array.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).