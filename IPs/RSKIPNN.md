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

With the introduction of the opcode of `CREATE2` now exists the possibility that, given a transaction list to be executed on the same block, there would be a transaction that destroys. This normally would not be a problem, but, in order to reduce modification times on the Unitrie storage, several cache levels were implemented. 

## Specification	

In order to prevent this problem, a special check had to be implemented. Basically, we keep track of the deleted accounts on previous transactions in the execution of the block and, on the remote case that a `CREATE2` is invoked with the same parameters than the one deleted, then the execution is reverted, and the result is the zero address. As with any error produced by a contract transaction, all gas is spent. 

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
