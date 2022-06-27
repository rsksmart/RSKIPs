---
rskip: 42
title: Remove world midstates from receiptsq
description: 
status: Adopted
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2017-06-22
---


# Remove world midstates from receipts

|RSKIP          |42           |
| :------------ |:-------------|
|**Title**      |Remove world midstates from receipts|
|**Created**    |22-JUN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted | 

# **Abstract**

The aim of this RSKIP is to remove the final world states from transaction receipts. Intermediate world states were included to receipts to be able to pinpoint a fraud transaction and prove it is fraudulent by sending a short message that requires low CPU processing to verify. It turns out that neither of these design requirements is met by the current RSK design.

## Discussion

Detecting an invalid final world state may require an enormous amount of original state data and final state merkle branches. For example, if a contract queries the balance of several other contracts, all the merkle branches corresponding to such queries must be included in the proof. as a block can hold a single transaction consuming all gas, the CPU consumption for a proof could be as high as the block gas limit. Therefore, an attacker willing to produce fraud blocks can force a node to essentially do almost a complete block validation. Removing the final states also have the benefit that the intermediate merkle hashes need not to be immediately computed, and can be computed at the end of transaction processing. Removing final world states from receipts reduce the complexity of fraud proofs considerably by turning a fraud proof into a block, plus an indicator that it is fraudulent. 

It is however desirable that the final world state is included in the header to allow nodes to check that the VM executions have yield the correct state, and to prevent a consensus problem to be detected only when the next block comes.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).