---
rskip: 59
title: Child Contracts
description: 
status: Rejected
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2016-06-11
---

# Child Contracts

|RSKIP          |59           |
| :------------ |:-------------|
|**Title**      |Child Contracts |
|**Created**    |11-JUN-16 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Rejected in favor of RSKIP125 |

# **Abstract**

Storage rent presents several challenges, one is how to pay the storage in a crowd-contact (a contract that doesn't have any special owner).
En example of such contract is a DNS-like contract where users register names. Another example is an ERC20 contract. 
One solution to this problem is to create a master contract that has child contracts. Child contracts are owned by individual users, and 
each child contract stores a portion of the storage that is strictly related to that user. 
When a user interacts with the master contract, the master contract calls the corresponding child contract, so the user is mostly 
interacting with the child contract, but not with all the remaining child contracts. For example, the transfer() method in an ERC20 contract 
would only call methods on the source and destination child contracts. This reduces storage rent and may also facilitate the 
parallelization of transaction execution. The main problem is that the master contract must store the child contract address in its own storage space, and therefore 
the storage rent consumed when calling the master contract will always be proportional the number of users registered in the master contract.
this RSKIP proposes that contracts can create "named" child contracts. The addresses of these contracts are chosen so that they can be re-computed with only an external address provided by the user.

# **Specification**

A new opcode CREATECHILD is created. The new CREATECHILD arguments are: value, inCodeOffset, inCodeSize, inAChildddressOffset.

The new argument inAddressOffset specifies an offset in memory where ChildAddress (a 20-byte value) is stored.

The address of the contract created will be sha3omit12(RLPList(SrcAddress,0,ChildAddress)). No that this address cannot collude (in practice) with a normal 
address because a normal address is constructed by the hash of a list of 2 fileds, while this list has 3 fields, and an RLP list specifies the number of elements. 

## Competing Proposals

It's important to note that the CREATE2 opcode defined [here]( https://github.com/ethereum/EIPs/blob/bd136e662fca4154787b44cded8d2a29b993be66/EIPS/abstraction.md)
has the same properties, so CREATE2 can be used insted of CREATECHILD. However CREATE2 specification strangely uses "+" for concatenating fields intead of the standard RLP encoding and this may create disastrous vulnerabilities. 

The definition of CREATE2 is the following:

New opcode at 0xfb, CREATE2, with 4 stack arguments (value, salt, mem_start, mem_size) which sets the creation address to sha3(sender + salt + sha3(init code)) % 2** 160, where salt is always represented as a 32-byte value.
By this definition, salt corresponds to the ChildAddress field.

## Other solutions

Another sotution to this problem in an ERC20 contract is to identify token owners by a sequential number related to the master contract nonce, which dictates what are the addresses of child contract created.
Therefore a token owner would be identified by a number local to the ERC20 contract intead of by a global address. An external global index can be created by token owners where they can map local indexes to 
global addresses. While this works, it has a high overhead: now the owners must pay rent on the index entry, and one entry must be added per token owned. Also 

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
