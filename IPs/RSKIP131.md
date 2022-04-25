---
rskip: 131
title: Preventing CREATE2-after-SUICIDE in the same block
description: 
status: Draft
purpose: Sca, Usa
author: SMS (@sebastians), SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2019-06-10
---

# Preventing CREATE2-after-SUICIDE in the same block

|RSKIP          |131           |
| :------------ |:-------------|
|**Title**      |Preventing CREATE2-after-SUICIDE in the same block |
|**Created**    |10-JUN-19 |
|**Author**     | SMS & SDL |
|**Purpose**    |Sca, Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

The purpose of this RSKIP is to forbid the creation of a contract that was destroyed in the same block.

## Motivation

With the introduction of the opcode `CREATE2` there exists the possibility that a transaction destroys a contract and afterwards another transaction creates a new contract on the same address, both in the same block. 

This does not pose any thoretical vulnerability, but in practice, handling this case efficiently requires the use of specific overcomplicated data structures. Modifying the state while supporting reversion (for OOG and REVERT) requires the use of caches (which store new storage values) or a journal (which stores previous storage values). Currently the RSK reference platform uses several nested caches. These caches operate as key/value maps, which record changes that are later applied to the Unitrie. Destroying a contract involves  removing the account nodes and all child nodes from the Unitrie, which contain code and contract addresses. In the cache, this would be achieved by nullifying all the subkeys starting from the account key. But this would require bringing into the cache all storage cell values at once. This operation is O(N) where N is the number of cells in the storage. However the cost of the SUICIDE opcode is constant, and does not account for the excessive I/O required to load every cell to delete. This seems unecesary since the actual removal of the account node and all child nodes from the Unitrie is a O(1) operation. This directly leads to a more efficient implementation of the cache where contracts to be deleted are marked for recursive deletion in a special map, but the actual child nodes are not nullyfied in the cache map. Any sub-key of a recursive deleted key is consider invalid and must not be accessed. Accesing storage cell keys requires checking first if the account was deleted or not, which is still a fast operation. 
So far so good. However, interleaving recursive deletion which creation invalidate the special map, and requires to fall back to the load all / write all functionality. This in turn may be used as a DoS attack vector.

Since contract sucidal is a very rare operation, and same contract creation after suicidal would still be an even more rare operation, it's much better to prevent this corner case that is complex to handle and error-prone.  
Therefore in this RSKIP we propose to add special logic to prevent the problem and fail the creation of contracts in the same block they were destroyed.

We note that the use of contract destoy and re-creation can be used to update contract ode. A contract owner may want to put the sucidal transaction back-to-back the re-creation transaction to prevent the target contract to be called when it's not avaiable. We argue that  this is not clean method for upgrading code and even if desired, the owner cannot fully control if the two transactions would occur in the same block and therefore this upgrade method is not atomic.

## Specification	

A special check is implemented on the contract creation logic. We keep track of the deleted contracts on previous transactions executed on the block and, if a `CREATE2` is invoked that would generate one of the deleted contracts addresses, then no code initialization is executed, and the result (pushed on the stack) is the zero. As with any error produced by a contract transaction, all gas is spent. 

## Backwards Compatibility

This RSKIP defines the behavior to be included with the `CREATE2` opcode, as specified in [RSKIP125](IPs/RSKIP125.md). All changes originated from this RSKIP are tied to the RSKIP125 and won't be implemented on its own.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
