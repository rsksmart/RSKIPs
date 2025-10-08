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

The purpose of the RSKIP is to document the change of the difficulty divisor for the calculation of the difficulty for the next block. The constant is changed from 1 over 50 to 1 over 400 in order to smooth the drop of difficulty in consecutive blocks in order to delay potential selfish mining attacks.

## Motivation

The decision to reduce the slope of the drop in difficulty was made in order to increment the number of blocks necessary to drive the block difficulty to nearly zero. This is useful to delay a potential attack to the SPV bridge with Ethereum. This attack could be possible by isolating the node and continuously reducing the difficulty of the blocks by now mining in an isolated competition-less chain, which could be done with a fraction of the hashing power with the old difficulty divisor. 

By doing this, you could form a correct chain with much less work than the original difficulty required and successfully orchestrate the attack. The hashing power necessary to do this would be a small fraction of the network instead of the 51%. With the new divisor, the attack proves to be much more complicated, and thus it means that a successful attack would need a almost half of the hashing power and more than 1400 correct blocks to achieve the same result (based on the research team calculations).

## Specification

The implementation is simple. A new constant is defined so we differentiate the initial difficulty divisor from the new one. When calculating the new difficulty and asked for the divisor, a check for the RSKIP is done, if the RSKIP is activated, then the new divisor is returned. 

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


