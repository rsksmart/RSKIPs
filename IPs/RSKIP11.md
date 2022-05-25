---
rskip: 11
title: TXINDEX Opcode
description: 
status: Adopted
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2016-08-07
---
# TXINDEX Opcode

|RSKIP          |11           |
| :------------ |:-------------|
|**Title**      |TXINDEX Opcode |
|**Created**    |07-AUG-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Adopted |

## Pre-git revisions

Modification 1.2:  Date: July 22, 2017

Modification 1.3:  Date: October 18, 2017

Revision: 1.3


# **Abstract**

This RSKIP defines a new opcode to obtain a the transaction index number. In the future this opcode can be used to re-direct a request (a contract method call) to a set of "child" contracts, to increase the parallelization factor in transaction processing, when the same service can be provided by several workers and when [RSKIP04]  (Parallel Execution using runtime contract dependencies) is implemented. 

# **Motivation**

There are several motivations to have a TXINDEX opcode. Here we present one. Consider the following example. A SERVER contract. The SERVER contract provides a service and receives a payment for this service. To avoid becoming a bottleneck, the SERVER contract avoids writing to a single persistent storage/value. Instead it diverts the payments by calling one of many a sub-contracts which actually receives the payment and perform the work requested. This requires the SERVER contract to randomly split user asset calls depending on some random property of the call. It cannot be done by maintaining a persistent nonce variable and incrementing it, because the sole action of incrementing it modifies the state and prevents parallelization. 

One way of doing it is by adding a NONCE opcode which returns the nonce of the transaction originating the payment. Therefore (block-id | msg.origin | msg.nonce) is an unique value that can be used for splitting. If a contract performs several calls to the same SERVER contract, the nonce will repeat, which is good. A simpler system creating a TXINDEX opcode that returns the transaction index in the block. This does not guarantee that incoming requests will be diverted equally between childs. For instance, suppose that the child index is chosen as (TXINDEX % 10). If there are 10 childs, the transactions that call the contract can have indexes 0,10,20,30 and 40, all of them will end up in the same slot.  However, statistically, as the number of transactions increase, they will end up filling the slots uniformly. 

A different approach is to have a per-contract counter, so each time a contract is called per transaction the counter increments.

# **Specification**

**TXINDEX**. Pushes on the stack the index of the current origin transaction. The cost of TXINDEX is 2.

[RSKIP04]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP04.md


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).