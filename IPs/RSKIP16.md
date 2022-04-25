---
rskip: 16
title: Combined State Tree 
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2016-11-01
---

# Combined State Tree

|RSKIP          |16           |
| :------------ |:-------------|
|**Title**      |Combined State Tree |
|**Created**    |01-NOV-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     |Draft |

## Pre-git revisions

Date: 06-09-2016

Revision: 3 (date 06-23-2017)

Status: Draft

# **Abstract**

The state of the RSK blockchain is split into several data structures. The account trie, which stores account records (balances, nonces, code hashes), the contract storage tries, which stores memory cells, and the smart contract code. This separation brings several problems. The main problem is garbage collection: to prevent the state storage to grow without limit, old and unused parts of the state must be removed (e.g. one month old). However doing this over several synchronized databases without interruption a full node is a requires a complex and error-prone garbage collecting algorithm. Therefore it’s highly beneficial if all the information is held in a single trie. Therefore a mark-and-sweep algorithm can remove unused parts. This RSKIP proposes a new structure for the world-state trie (called **Unitrie**), combining accounts, contracts, code  and storage, and enabling many future improvements.

# **Motivation**

The current RSK trie (inherited from Ethereum) has several problems, both in its design and in its implementation. The world-state consist of the state trie, plus the contract storage tries, plus the contracts codes. Each of these data structures must be  managed by an appropriate cache system that decides when to keep the data in memory, when to send it to SSD disk or even HDD disk. This multiplicity of caches of different kinds complicates the design of the platform node considerably. The same complexity occurs in garbage collection, the removal of unused parts of the state, and in state sync.
The current state trie provides no space to store accounts controlled by a different signature algorithm (e.g. RSA) or a different address hash digest length (e.g. 32 bytes). Also the state does not provide a namespace to store other state information different from the accounts. For example, storing a Merkle Mountain Range (MMR) for past block commitments can help light clients synchronize faster by using compact SPV proofs.


# **Specification**

We merge the contract storage memory and account/contract memory into a single tree.

Changes:

1. The trie key is prefixed by a single byte 0x00, which is a namespace identifier. It's current value is zero.
2. For each account node, there is an optional Account sub-tree.  There are several fields in this tree. Each field is identified by a single byte fieldSelector. This byte is not "randomized" by a hash:
 * 0x00: Storage cells
 * 0x80: Code

Storage Cell keys are also randomized by a hash function application.

## Trie Path Components 

Each key in the trie is split into the four parts:

* Account Type & Namespace [1 byte, 1st bit is the namespace]
* Account address [20 bytes]
* Account fields Selector [1 byte, only in an account sub-tree]
* Storage Address [32 bytes, only for contract storage cells]

Account address and storage addresses are hashed with sha3 in the secure trie. 

 

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
