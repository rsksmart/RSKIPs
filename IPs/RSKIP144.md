---
rskip: 144
title: Parallel Transaction Execution for Unitrie	
description: 
status: 
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 3
created: 2019-10-23
---
|RSKIP          |144           |
| :------------ |:-------------|
|**Title**      |Parallel Transaction Execution for Unitrie |
|**Created**    |23-OCT-2019 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     | |

# **Abstract**

This RSKIP describes how miners partition transactions into disjoint sets and how full nodes should process transactions in order to be safely parallelized. 

# **Motivation**

RSK processes transactions from blocks one by one, in the specified order. This is because the final state after processing two transactions when applied in different order may differ. However most transactions do not use the same keys of the state and therefore they could be parallelized without interference.

There are several obstacles to parallelization. [RSKIP02] and [RSKIP04] explore different methods that worked prior the implementation of the Unitrie. 

This RSKIP propose using a runtime method to partition the transaction set into threads similar to RSKIP04 but tailored for the Unitrie. Miners are forced to serialize transaction execution and at the same time discover runtime key-access overlaps. Once all transactions have been processed and the partition is created, an index is created holding the first transaction number for each thread (the "partition" field).  ull nodes can use this index to split the transaction set and parallelize execution.


# Specification

A new field partitionEnds is added to the block header. partitionEnds contains an array of integers, indicating at what offset each partition ends. For example if partitionEnds is [2,4] then the first partition contains the transaction at offsets [0,1,2] and the second at offsets [3,4]. Values in partitionEnds must be ascending. If the block has no transactions, then partitionEnds must be empty. The maximum number of threads the miner can specify is 16.

Full nodes must use the partitionEnds field to split the transaction set and parallelize execution. During execution, when a state key is read, it is marked with the thread index in a per-thread readMap. When a key is written, the key is marked in a per-partition writeMap map. When all transactions of a certain thread have been processed, the writeMap is scanned. For every entry, the new value is compared with the previous value (before the partition transactions were executed).  If it didn't change, then the entry is removed from the writeMap. When all threads have finished processing, the readMaps and writeMaps are "merged". This requires efficient maps that enable traversing the keys in ascending lexicographic order. If a key belongs to a writeMap and a readMap of another thread, then the block is considered invalid. If a key belongs to a writeMap and a writeMap of another thread, then the block is also considered invalid. Recursive deletes must be correctly and efficiently handled. Miners are incentivized to produce a valid and efficient partition because by doing so their blocks spread faster over the network. It's also possible in the future to use a per-thread gas limit, instead of a global gas limit.

When a miner executes transactions to create an (unsolved) block, the miner must execute the transactions serially and decide which thread the transaction belongs to. For each thread the miner creates, it maintains the last world state (i.e. as a cache of changes) and a per-thread read/write map (readMap/writeMap).  

Also the miner maintains a global writeMap/readMap that maps a key to the thread that wrote it or last thread that read it (optionally it could map to all threads that read it). This is maintained for efficiency purposes, as it can be dynamically built from the per-thread maps.

The new transaction is run to be included in the template block assuming it runs on a new thread (initial block state), and a new per-thread readMap/writeMap is built. After it is executed, the transaction writeMap is scanned for changes (as described before) and keys may be removed accordingly. Then the transaction read/write maps are compared with the global read/write maps. If the transaction does not read any globally written key, nor writes a globally read key, then this transaction is assigned to a new thread, if available, and the global maps are updated to include the new thread reads and writes. Otherwise, the transaction is assigned to one of the previous threads. If the transaction reads a key that has been written, then it is assigned to the thread that wrote it (as specified in the global writeMap) into the first place of the threads queue, therefore it will not affect the outcome of the remaining transactions of the thread. If it writes a key that is already read, it should be assigned to one of the threads that read it (i.e. the last). In that case, it must be assigned as the last element of the thread queue, so it cannot change the outcome of the previous transaction. If there are more than one thread that has read the aforementioned key, the thread can be selected by some efficiency criteria by the miner (i.e. the threads with lower gas consumption up to that point). Both criteria are satisfied and for equal or distinct keys (read maps requires the transaction to be assigned to one threads but write maps requires it to be assigned to a different one), then all threads must be joined by the following procedure :

- When joining the threads, all the "read key" threads transactions must go first, then the new transaction and finally all the "write key" threads transactions. By doing so the inserted transaction cannot change the outcome of any other one.

There can be the case of the existence of a loop. If there is no loop the maps and final thread states can be simply efficiently joined. 

Let's analyze a case of a loop:

- Thread 1: Tx1=(R0,W1)
- Thread 2: Tx2=(R0,W2)
- Thread 3: Tx3=(R1 R2, W0)

(Rn means read key n, and Wn means writes key n) 

When Tx3 is executed in a separate thread, it becomes incompatible with any serialization of the threads. In this case the miner can opt to remove the transaction from the block or try to assigned it to a preexistent thread (always as the last thread). If it opts to force an assignment then the previous execution (in a separate thread) is completely discarded, and the transaction is re-executed in the context of the selected thread last state, and the thread readMap/writeMap is updated accordingly.  It will be the case that the new context changes the thread read/write maps. Therefore the criteria to join threads must be re-checked and more threads may need to be re-executed on that same thread.  To prevent this cascade effect, is is recommended that the transaction is excluded from the block. The sender may want to increase the failed transaction gasPrice so that the transaction is picked first next time, and it is not forever blocked.

Always at the end of processing a transaction, the new reads/writes are copied to the global write/reads maps for efficiency.  This procedure repeats until there is no more gas left to consume or there are no more valid transactions to add. 

Special treatment may be needed for recursive deletes, as used in suicides. 

## Transaction Censorship

For a target (victim's) transaction tx0, an attacker could try to broadcast transactions tx1..txN with high gasPrice so that they are chosen first, and so that tx0 is discarded because it would build a loop in the access maps and require re-execution. If the default mining behavour is to discard the transaction, then this would allow censorship at low costs. If the transaction tx0 is postponed and re-executed in following blocks, then it also possible that the miner needs to re-execute it an unbounded number of times without succes, which impacts the miner's operation cost.
Therefore we suggest than once every k blocks (i.e. k=5) a block must be serial (a "serial block"), and the miner cannot specify any parallelism in the serial block. This will enable all postponed transactions that could not be parallelized to compete and be included in serial blocks. An alternative is to pseudo-randomize the choice of the serial block (i.e depending on the previous block hash). Adding this uncertanity however doesn't seem to provide any benefit.

## DoS Security

A user could build a transaction that transfers tiny amounts of RBTC to many contracts or executes EXTCODEHASH/EXTCODESIZE/BALANCE to prevent the partitioning algorithm to produce efficient partitions. However these opcodes have a high cost in gas. If this problem is still present, then the cost of these opcodes may need to be increased. 

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
