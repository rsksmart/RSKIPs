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

The purpose of this IP is to explain the logic behind the decision made to forbid creating a contract that was destroyed on the same block, and how this was implemented.

## Motivation

With the introduction of the opcode of `CREATE2` now exists the possibility that, given a transaction list to be executed on the same block, there would be a transaction that destroys. This normally would not be a problem, but, in order to reduce modification times on the Unitrie storage, several cache levels were implemented.

## Specification



### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
