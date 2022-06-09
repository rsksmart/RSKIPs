
|RSKIP          |144           |
| :------------ |:-------------|
|**Title**      |Parallel Transaction Execution for Unitrie |
|**Created**    |23-OCT-2019 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |3 |
|**Status**     | |

# Abstract

This RSKIP describes how miners partition transactions into disjoint sets in order to be safely parallelized, and how full nodes should process transactions.

# Motivation

Parallelizing the execution of transactions allows increasing the block gas limit without increasing the block execution time, and improves the scalability of RSK by increasing the transaction throughput.

Now, RSK nodes process transactions from blocks one by one, in the specified order. This is because the final state after processing two transactions when applied in different order may differ. However most transactions do not use the same keys of the state and therefore they could be parallelized without interference.

There are several obstacles to parallelization. [RSKIP02](RSKIP02) and [RSKIP04](RSKIP04) explore different methods that worked prior the implementation of the Unitrie. This RSKIP proposes using a runtime method to partition the transaction set into threads similar to RSKIP04 but tailored for the Unitrie.

Miners are forced to serialize transaction execution to create blocks. At the same time they execute the transactions, they discover runtime key-access overlaps between transactions and build an execution plan that is included in the block header. For a simpler overlap detection and to prevent DoS attacks, an additional part is added including all the transactions that could not be parallelized, that is executed after the execution of the parallel parts is completed. Once all transactions have been processed, the partition is created along with a schedule that determines which transactions belong in each part.

Full nodes can use this schedule to split the transaction set and parallelize execution.

# Specification

Transactions in a block are divided into `N+1` partitions, where the first `N` partitions are executed in parallel and the last partition is executed after the others. We refer to the first `N` partitions as the _parallel partitions_ and to the last partition as the _sequential partition_.

<p align="middle">
  <img src="./RSKIP144/schedule.png" alt="Schedule" /><br />
  <em>Block with 3+1 partitions. The 3 partitions are run in parallel, and after completition of all, the 4th partition is run.</em>
</p>

## New block header field

A new field `partitionEnds` is added to the block header. It determines how transactions are partitioned in a block.

A new constant `maxTransactionExecutionThreads` is specified. It determines the minimum number of cores required to run the RSK node. Initially, `maxTransactionExecutionThreads = 4`

This field consists of an array of short unsigned integers that indicates at which position in the transaction list each partition ends. For example, in a block with 10 transactions, `partitionEnds = [3, 6]` indicates that the first _parallel partition_ contains transactions 0, 1 and 2; the second _parallel partition_ contains transactions 3, 4 and 5; and the _sequential partition_ contains transactions 6 to 9.

- Values in `partitionsEnd` must be greater than 0 and in ascending order.
- An empty `partitionEnds` indicates that all transactions go in the _sequential partition_.
- The maximum number of parallel partitions that the miner can specify is equal to `maxTransactionExecutionThreads`.
- The REMASC transaction must be included as the last transaction of the sequential partition.


## New block validation consensus

It must be ensured that blocks that have transactions that are executed in parallel always produce the same output. Therefore, when a block is executed in parallel, nodes must verify the resulting world state is deterministic.

For simplicity, three types of connections between transactions are identified:
- Both transactions write the same storage key
- One transaction reads a key that the other transaction writes

Any pair of transactions with any of these characteristics cannot be in different parallel partitions. If so, the block must be rejected.

> Transactions that are from the same sender account are considered connected because they modify the nonce

This cases should not be considered as writing a key, since they doesn't affect the output of the parallel execution:
- Adding 0 balance
- Write a key with the value that was already set

### Block validation algorithm

A `readMap` and a `writeMap` is created per _parallel partition_. During execution of each partition,
- when a storage key is read, it is marked in the `readMap`,
- when a key is written, the key is marked in the `writeMap`.

When all parallel partitions have finished processing, the `readMap`s and `writeMap`s are scanned to find connections between transactions of different threads. This requires efficient maps that enable traversing the keys in ascending lexicographic order.
- If a key belongs to a `writeMap` and a `readMap` of another _parallel partition_, then the block is considered invalid.
- If a key belongs to a `writeMap` and a `writeMap` of another _parallel partition_, then the block is also considered invalid.

The _sequential partition_ does not need any new validation.

# Suggested miner implementation

When a miner executes transactions to create an (unsolved) block, the miner must execute the transactions serially and decide which partition the transaction belongs to. The block gas limit is replaced by a per-partition gas limit.

Following the connection types between transactions described above, a transaction is connected to a _parallel partition_ if it is connected to any transaction in that partition. The miner maintains a `writeMap` and a `readMap` that store which partition/s write or read each storage key. The miner then executes transactions from the pool and keeps track of which storage keys the transaction reads and writes. After executing the transaction, the miner compares the transaction read and written keys with the `readMap` and `writeMap`. The following scenarios are possible:

1. The transaction is not connected to any existing partition:
    1. The miner assigns the transaction to an empty _parallel partition_
    2. If no empty partition exists, the miner assigns the transaction to the less full _parallel partition_
    3. If all _parallel partitions_ are full, the miner assigns the transaction to the _sequential partition_
2. The transaction is connected to one _parallel partition_:
    1. The miner assigns the transaction to the _parallel partition_
    2. If the _parallel partition_ is full, the miner assigns the transaction to the _sequential partition_
3. The transaction is connected to more than one _parallel partitions_:
    1. The miner assigns the transaction to the _sequential partition_
    2. Alternatively, the miner might merge the connected partitions and assign the transaction there

After assigning a transaction to a partition, the miner updates the `readMap` and `writeMap` accordingly. When no more transactions can be included in the block, the miner executes the block in parallel to produce the final state.

To prevent DoS attacks, miners only execute a transaction if it fits in the sequential set. In this way, miners are guaranteed to include the transaction in the block regardless of the read and written storage keys.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
