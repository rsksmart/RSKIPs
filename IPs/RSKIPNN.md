# Creating a contract destroyed on the same block

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

Prevents the creation of a contract with the same address of a contract that was destroyed previously in the same block execution. In order to prevent cache inconsistencies across the different cache levels.  

## Specification



### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
