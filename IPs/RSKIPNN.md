# On creating a contract destroyed on the same block

|RSKIP          |NN           |
| :------------ |:-------------|
|**Title**      |Creating a contract destroyed on the same block |
|**Created**    |10-JUN-19 |
|**Author**     | SMS |
|**Purpose**    |Sca, Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

The purpose of this RSKIP is to explain the logic behind the decision made to forbid creating a contract that was destroyed on the same block, and how this was implemented.

## Motivation

With the introduction of the opcode `CREATE2` exists the possibility that, given a transaction list to be executed on a block, there would be a transaction that destroys a contract and afterwards another transaction creates a new contract on the same address. 

This normally would not be a problem, but, in order to reduce accesses on the Unitrie storage, several cache levels were implemented. With this cache implementation the Unitrie is not touched until all transactions are finished. This would mean that, in our problem, there is in the Unitrie an address with all its code and storage, and in the cache there are two operations, one of deletion of this address and the second creation. 

So, what should happen when we commit these changes? Should we consider the order of the operations? Should we make a special check on the commit phase for this case? It should be noted that this use case is both rare and expensive, given the contract creation that it involves (all called from the same address). 

Considering all these questions we thought it was better to not change anything regarding the Unitrie implementation and all its cache levels, and put this logic on the place where the problem first occurs, the *contract creation*.

## Specification	

In order to prevent this problem, a special check had to be implemented on the contract creation logic. Basically, we keep track of the deleted accounts on previous transactions executed on the block and, on the remote case that a `CREATE2` is invoked with the same parameters than the one deleted, then the execution is reverted, and the result is the zero address. As with any error produced by a contract transaction, all gas is spent. 

## Backwards Compatibility

This RSKIP defines the behavior to be included with the `CREATE2` opcode, as specified in RSKIP125. All changes originated from this RSKIP are tied to the RSKIP125 and won't be implemented on its own.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
