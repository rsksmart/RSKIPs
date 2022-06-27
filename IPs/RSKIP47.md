---
rskip: 47
title: CALLNUM opcode
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2017-10-18
---

# CALLNUM opcode

|RSKIP          |47           |
| :------------ |:-------------|
|**Title**      |CALLNUM opcode|
|**Created**    |18-OCT-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

# **Abstract**

This RSKIP defines two a opcodes to obtain incremental counter per contract without the need to write to a contracts persistent memory. In the future it will be important to re-direct a request (a contract method call) to a set of "child" contracts, to increase the parallelization factor, when the same service can be provided by several workers and when [RSKIP03]  (Parallel Execution using static contract dependencies) is implemented. . To redirect a request, the VM can provide a nonce value (at no cost). This nonce can be the index of the transaction being processed (as defined in the [RSKIP11], TXINDEX opcode) but this provides only statistical spread to child workers, but not guaranteed spread. This RSKIP defines the opcode CALLNUM, which retrieves the number of times the contract has been called, but where the counter only increments the first time a contract is called in an externalk  transaction

# **Motivation**

Consider the following example. A SERVER contract. The SERVER contract provides a service and receives a payment for this service. To avoid becoming a bottleneck, the SERVER contract avoids writing to a single persistent storage/value. Instead it diverts the payments by calling one of many a sub-contracts which actually receives the payment and perform the work requested. This requires the SERVER contract to randomly split user asset calls depending on some random property of the call. It cannot be done by maintaining a persistent nonce variable and incrementing it, because the sole action of incrementing it modified the state and prevents parallelization. 

An opcode CALLNUM returns the number of calls that have been done in the block where only the first call in an external transaction incremements the counter. For example, if transaction 1 calls a contract 5 times, CALLNUM will return 0 all five times. IF transaction 10 calls the same contract 3 times, and no other prior transaction has called it, then CALLNUM will return one three times. 

# **Specification**

**CALLNUM**. Pushes on the stack the number of CALLs that this origin transaction has made so far, counting only the first of each external transaction. The first call receives the value 0. If a CALL is reverted because of an out-of-gas exception, the CALLNUM increment is **not** reverted. The cost of CALLNUM is 2. The CALLNUM counters are **not** stored between blocks.

[RSKIP03]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP03.md
[RSKIP11]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP11.md

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).