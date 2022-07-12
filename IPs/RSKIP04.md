---
rskip: 4
title: Parallel Execution using runtime contract dependencies 
description: This RSKIP describes how miners partition transactions into disjoint sets and how full nodes should process transactions in order to be safely parallelized. 
status: Accepted
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-06-22
---

#Parallel Execution using runtime contract dependencies

|RSKIP          |04           |
| :------------ |:-------------|
|**Title**      |Parallel Execution using runtime contract dependencies |
|**Created**    |22-JUN-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Accepted |

# **Abstract**

This RSKIP describes how miners partition transactions into disjoint sets and how full nodes should process transactions in order to be safely parallelized. 

# **Motivation**

RSK processes transactions from blocks one by one, in the specified order. This is because the final state after processing two transactions when applied in different order may differ. However most transactions do not use the same Accounts/Contracts and therefore they could be parallelized without interference.

There are several obstacles to parallelization. [RSKIP02] allows contracts to specify dynamic dependencies. Therefore it is possible to partition the transactions into disjoint sets. However creating and/or updating such dependency graphs is expensive. Call loops in the dependency graphs also make dependency exploring more error-prone. The algorithm that creates or updates the dependency graph can therefore be target of DoS attacks. 

Another problem of using static dependencies is that the partition has to be made taking into account all possible execution paths and all possible calls, where in reality only few external contracts may be called for a transaction. Therefore this reduces the parallelization.

This RSKIP propose using a runtime method to partition the transaction set. Miners are forced to serialize transaction execution and at the same time discover runtime overlaps. Once all transactions have been processed and the partition is created, an index is created holding the first transaction number for each subset (partition field).  The hash of the partition field is stored in the block header. Full nodes can use this index to split the transaction set and parallelize execution. During execution, each called contract is marked with the index of the subset. If at any moment a contract is to be marked twice with different indexes, the block is considered invalid. To prevent miners from not correctly specifying the partition, if during the transaction processing a node finds that a transaction in the same partition subset does not overlap with a contract from the subset, the block is considered invalid.

## Word states in receipts

As every transaction subset starts with the same world state, it is not immediately possible to deduct the final world state from the last world states stored in the last transaction receipts of each subset. Therefore an SPV wallet cannot easily receive proof that the final world state is invalid. There are two possible remedies: try to solve the problem, or rethink the original design choice of including the final world state in receipts. 

To solve the problem, instead of storing the final world state, receipts should store the hash of the cache tree that stores the differences between states. A proof that the final differences can be merged into the final state would consist of: Branches of the original state tree, all the difference branches, branches of the final state tree. This proof could be rather large.

Another option is to remove the final world states from transaction receipts [RSKIP42]. Intermediate world states were included to receipts to be able to pinpoint a fraud transaction and prove it is fraudulent by sending a short message that requires low CPU processing to verify. It turns out that neither of these design requirements is met by the current RSK design. Detecting an invalid final world state may require an enormous amount of original state data and final state merkle branches. For example, if a contract queries the balance of several other contracts, all the merkle branches corresponding to such queries must be included in the proof. as a block can hold a single transaction consuming all gas, the CPU consumption for a proof could be as high as the block gas limit. Therefore, an attacker willing to produce fraud blocks can force a node to essentially do almost a complete block validation. Removing the final states also have the benefit that the intermediate merkle hashes need not to be immediately computed, and can be computed at the end of transaction processing. Removing final world states from receipts reduce the complexity of fraud proofs considerably by turning a fraud proof into a block, plus an indicator that it is fraudulent. 

It is however desirable that the final world state is included in the header to allow nodes to check that the VM executions have yield the correct state, and to prevent a consensus problem to be detected only when the next block comes.

# Specification

A new field partitionEnds is added to the block header. partitionEnds contains an array of integers, indicating at what offset each partition ends. For example if partitionEnds is [ 2,4] then the first partition contains the transaction at offsets [0,1,2] and the second at offsets [3,4]. Values in partitionEnds must be ascending. If the block has no transactions, then partitionEnds must be empty.

When a miner executes transactions to create an (unsolved) block, the miner does the following:

**Definitions:** 

A Partition (class) contains:

1. list :  a lists of transactions 

2. touchedContracts: an empty contract list (pointers to actual contract objects)

3. Order: an integer that specifies the order

**Algorithm:**

1. All contracts hold a "partition" object reference, initially set to nil. Also all contracts have a boolean flag “ReadOnly” initially set to true.

2. Let partitions be a list of Partition objects. Let suicided be a list of contracts.

3. Repeat until there are no more transactions

    1. Append a new Partition to partitions. Let currentPartition be the last appended partition. Let currentPartition.order be a monotonically increasing number.

    2. Let conflictPartitions be an empty set of Partition object references.

    3. Take the next available transaction. 

    4. Execute the transaction. For every CALL or CALLCODE to dst_contract executed:

        1. If dst_contract does not exist and it’s not in the suicides list, proceed as normal and skip further checks.

        2. If dst_contract is in the suicides list, add dst_contract.partition to conflictPartitions list, and proceed as normal skipping further checks.

        3. If call is type CALLCODE (not CALL) skip further checks.

        4. If dst_contract writes to persistent centralized memory, clear the ReadOnly flag.

        5. If dst_contract  suicides, clear the ReadOnly flag and add dst_contract to the suicides list.

        6. If dst_contract  writes to distributed memory (RSKIP01) do:

            1.  Read the destination dmem_contract.partition

            2. if dmem_contract.partition is nil, set it to currentPartition and add the contract to currentPartition.touchedContracts. 

            3. If dmem_contract.partition is different from currentPartition and not nil, add dmem_contract.partition contract to conflictPartitions list.

        7. When the CALL returns, if dst_contract.partition is nil and ReadOnly is false, set it to currentPartition and add dst_contract to currentPartition.touchedContracts. 

        8. If dst_contract.partition is different from currentPartition and not nil, add dst_contract.partition contract to conflictPartitions list. 

    5. If conflictPartitions is not empty:

        9. Sort the partitions in conflictPartitions by partition order.

        10. Let targetPartition be the first partition in conflictPartitions.

        11. For every partition p (in ascending sequence) in conflictPartitions  do 

            4. Execute Merge(p, targetPartition)

        12. For every other partition p execute Merge(p,targetPartition)

        13. Remove the last element from partitions

        14. Set currentPartition to targetPartition

    6. Insert current transaction in currentPartition.list

4. Let c = 0

5. For every partition p in partitions

    7. if p.list is not empty then

        15. append p.list to block.list

        16. append block.list.size to block.partitionEnds

**Subroutines:**

Merge(fromPartition,toPartition)

1. If fromPartition==toPartition then return

2. Mark every contract c in fromPartition.touchedContracts with c.partition = toPartition.

3. Append every contract in fromPartition.touchedContracts into toPartition.touchedContracts 

4. Append every transaction in fromPartition.list into toPartition.list preserving order.

The worst case of this algorithm is when each partition goes through a waterfall of marges. 

This example shows the contracts touched by each transaction, and the partitions created (a subset in the left means it was created from a merge operation later).

0: [ 0 ] [ 0 1 2  3 4 5]

1: [ 1 ] [ 1 2  3 4 5] []

2: [ 2 ] [ 2  3 4 5] []

3: [ 3 ] [ 3 4 5] []

4: [ 4 ] [ 4 5 ] []

5: [ 4 5 ] []

6: [ 3 4 ] 

7: [ 2 3 ] 

8: [ 1 2]

9: [ 0 1]

The complexity is O(N^2).

## Other VM Modifications

In RSK contracts are created and can be immediately used by following transactions. Also contracts can execute other contracts during initialization. This brings problems, such as in this example:

T0: Calls contract A

T1: Creates contract A

T2: Calls contract A

The call to contract A will fail and it’s not affected by T1. T2 will be associated with the partition of T1 correctly.

Now if T0 and T1 are executed in parallel, it’s possible that T1 is executed first, so that T0 now calls an existing contract A instead of failing. 

Therefore parallel executions should not share a single state tree, but should cache state modifications (contract creations and removals) in a  per-partition cache. After all partitions have been executed, the caches should be merged into the state tree preserving the order.

 

A different thing happens with suicides. In RSK suicides are made effective at the end of the contract processing. Let’s see an example:

T0: Calls contract A

T1: Calls contract A, then A suicides.

T2: Calls contract A

When T2 is executed, contract A no longer exists, so T2 cannot be merged with the partition having T0 and T1. There two are possible solutions: either the VM is modified such that suicides only take effect when all transactions have been processed, or we modify the partitioning code so that suicided contracts are added to a suicided list. Each time a suicided contract is called, although the effect of suicide call remains intact, the transaction is merged into the partition of the suicided contract. Full nodes must also maintain a suicides list to detect invalid partitions due to suicide interactions.

## DoS Security

A user could build a transaction that calls almost all contracts to prevent partitioning. However calling contract has an expensive gas cost. If not, then calling cost should be adjusted to prevent this attack.

## Interaction with RSKIP01

RSKIP01 defines distributed storage with the aim of reducing contract/wallet dependencies in centralized contracts and allow execution parallelism. To provide this parallelism for contracts the partitioning algorithm allow overlaps on contracts that do not write to persistent memory. A  "ReadOnly" flag is used to detect when a contract perform persistent centralized memory writes. Calls to contracts that are ReadOnly can be commuted as there are no side-effects. 

Writes to decentralized memory are treated similar to contract calls. Consider the following example:

T0: Message to contract A "Transfer X USD from account A to account B". 

T1: Message to contract A "Transfer X USD from account D to account E". 

If A does not use centralized memory for the method "Transfer", then the touched contracts during T0 execution are only A and B, while the touches contracts for T1 are only D and E. Therefore T0 and T1 can be safely parallelized.


[RSKIP02]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP02.md
[RSKIP42]: https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP42.md

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).