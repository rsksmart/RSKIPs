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

The purpose of the RSKIP is to document the change of the difficulty divisor for the calculation of the difficulty for the next block. The constant is changed from 1 over 50 to 1 over 400 in order to smooth the drop of difficulty in consecutive applications of this divisor to prevent selfish mining attacks.

## Motivation

The decision to reduce de slope of the drop in difficulty was made in orden to increment the window of attack to the SPV brige with Ethereum by the means of a classic selfish mining attack. This attack was posible by isolating the node and continously reducing the difficulty of the blocks, which could be done with a fraction of the hashing power with the old difficulty divisor. By doing this, you could form a correct chain with much less work than the original difficulty required and succesfully orchestrate the attack with a hashing power in the order of 10% of the total and up to 70 blocks to drive the difficulty to almost zero. With the new divisor, the attack proves to be much more complicated, and thus it means that a succesfull attack would need a whole mining pool of hashing power a more than 1400 correct blocks to achieve the same thing.

## Specification

The implementation is simple. A new constant is defined so we differenciate the initial difficulty divisor from the new one on the class Constants. When calculating the new diffculty and asked for the divisor, a check for the RSKIP is donde, if the RSKIP is activated, then the new divisor is returned. 

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
