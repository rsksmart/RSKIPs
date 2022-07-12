---
rskip: 3
title: Parallel Execution using static contract dependencies
description: This RSKIP describes how transactions should be processed in order to be safely parallelized.
status: Rejected
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-06-22
---


# Parallel Execution using static contract dependencies

|RSKIP          |03           |
| :------------ |:-------------|
|**Title**      |Parallel Execution using static contract dependencies |
|**Created**    |22-06-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Rejected |


# **Abstract**

This RSKIP describes how transactions should be processed in order to be safely parallelized. This RSKIP requires [RSKIP02]  (Dynamic Contract Dependency) to be deployed.

# **Motivation**

RSK processes transactions from blocks one by one, in the specified order. This is because the effect of two random transactions can differ if applied in different orders. However most transactions do not use the same Accounts/Contracts and therefore they could be parallelized without interference.

There are several obstacles to parallelization. [RSKIP02] allows contracts to specify dynamic dependencies. Therefore it is possible to partition the transactions into disjoint sets. 

The transaction partitioning could be made deterministic if contract relations were to be defined prior execution. Two new opcodes are proposed in order to define and undefine such relations. The effect of change in a relation only applies in the next block. Therefore contracts can still communicate with any other contract providing those contracts register callback addresses before the callback addresses are actually called.

# **Specification**

Prior execution, transactions are partitioned in dependency disjoint sets. This partition is based on a deterministic flood fill coloring algorithm. The algorithm is the following:

1. Set CurrentColor to zero. All transactions begin unassigned.

2. Start with the first unassigned transaction.

3. Take the source account/contract and color it with CurrentColor. Explore recursively all dependencies (dependent contracts must be also explored) and color them all with CurrentColor.

4. Take the destination account/contract and color it with CurrentColor explore recursively all dependencies and color them all with CurrentColor.

5. Increase CurrentColor

The main problem with this algorithm is: how to reduce the computations required for recursive coloring? One possibility is that dependencies are stored in Set data structures that allow fast join operation.

We use a [Treap] data structure.

Every contract stores the dependencies in a treap.

Every time we visit a contract, we visit all chids, and build treaps for them, and then we create the union of every child treaps. If we arrive at a already visited contract, we skip it but we return the reference of this contract in a resulting RefTreap structure. This is to track recursive references.

Add the treap to an ordered tree of treaps, where the key is the treap size. We also reference this resulting treap by the contract itself, to To minimize treap union time, we begin with the smaller treaps, join them, and put them back into the tree of treaps, ordered by size.

We repeat this process until a single treap is in the root.


[RSKIP02]:https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP02.md
[Treap]:https://en.wikipedia.org/wiki/Treap

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).