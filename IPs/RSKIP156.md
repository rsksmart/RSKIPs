# Increasing Difficulty Divisor

|RSKIP          |156           |
| :------------ |:-------------|
|**Title**      |Increasing Difficulty Divisor |
|**Created**    |30-JAN-20 |
|**Author**     |SMS |
|**Purpose**    |Sca,USa |
|**Layer**      |Core |
|**Complexity** |1|
|**Status**     |Draft |

## Abstract

The purpose of the RSKIP is to document the change of the difficulty divisor for the calculation of the difficulty for the next block. The constant is changed from 1 over 50 to 1 over 400 in order to smooth the drop of difficulty in consecutive applications of this divisor.

## Motivation

The decision to reduce de slope of the drop in difficulty was made in orden to increment the window of attack to the SPV brige with Ethereum. This attack was posible by isolating the node and reducing the difficulty of the blocks, which could be done with a fraction of the hashing power. 

## Specification

The implementation is simple. A new constant is defined so we differenciate the initial difficulty divisor from the new one on the class Constants. When calculating the new diffculty and asked for the divisor, a check for the RSKIP is donde, if the RSKIP is activated, then the new divisor is returned. 

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
