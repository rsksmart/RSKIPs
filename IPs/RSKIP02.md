---
rskip: 2
title: Dynamic Contract Dependency 
description: This RSKIP describes new opcodes to enable contracts to declare and enable dependencies with other contracts. 
status: Rejected
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2011-06-16
---

# Dynamic Contract Dependency

|RSKIP          |2           |
| :------------ |:-------------|
|**Title**      |Dynamic Contract Dependency |
|**Created**    |11-JUN-16 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Rejected|

# **Abstract**

This RSKIP describes new opcodes to enable contracts to declare and enable dependencies with other contracts. This will allow the partitioning of the transaction set into non-overlapping subsets to process them in parallel.

Rejected, in favour of [RSKIP04] .

# **Motivation**

RSK processes transactions from blocks one by one, in the specified order. This is because the effect of two random transactions can differ if applied in different orders. However most transactions do not use the same accounts/contracts and therefore their execution could be parallelized without interference.
There are several obstacles to parallelization. First, the dependencies are unknown until the transaction is actually executed. This implies that in case of an overlap in the set of modified accounts, transactions must be reverted and execution postponed. This optimistic execution can work on the average case, but can also be used as a vector to attack a node by DoS.
The transaction partitioning could be made deterministic if contract relations were to be defined prior execution. Two new opcodes are proposed in order to define and undefine such relations. The effect of change in a relation only applies in the next block. Therefore contracts can still communicate with any other contract providing those contracts register callback addresses before the callback addresses are actually called.

# **Specification**

### ADD_CALL_RELATION
Arguments: &lt;address&gt;

Cost: TBD

Add a new relation to this contract.

### REMOVE_CALL_RELATION
Arguments: &lt;address&gt;

Cost: TBD

Removes a relation from this contract.

### IS_CALL_RELATED
Arguments: &lt;address&gt;

Cost: TBD

Returns True if there is a relation, or False otherwise.


[RSKIP04]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP04.md

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).