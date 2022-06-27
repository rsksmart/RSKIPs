---
rskip: 64
title: Garbage Collector for State Pruning
description: 
status: Draft
purpose: Sca, Usa
author: SDL (@sergiodemianlerner), MMA
layer: Core
complexity: 2
created: 2018
---

# Garbage Collector for State Pruning

| RSKIP          | 64                                  |
| :------------- | :---------------------------------- |
| **Title**      | Garbage Collector for State Pruning |
| **Created**    | 2018                                |
| **Author**     | SDL & MMa                           |
| **Purpose**    | Sca, Usa                            |
| **Layer**      | Core                                |
| **Complexity** | 2                                   |
| **Status**     | Draft                               |

## Abstract

Stateful blockchains continuously generate state data. This data becomes useless when it is unreferenced by the best block and old enough that the node won't expect reverts of such depth. Also synchronized peers shouldn't request old state data and old data is not required to mine new blocks. This RSKIP proposes a method to reduce storage usage with very low overhead.

## Motivation

Currently every RSK full node keeps all historical data about the state after each block generated. After thousands of confirmation blocks, there is no need to access historical and overwritten state data while deep reorganizations become unfeasible. This historical state data consumes storage while it could be regenerated if needed.  It's clearly desirable to discard this state data in full nodes to reduce resource consumption and full node cost.


## Specification

Each block will be assigned an epoch based on its height. Every N=20000 RSK blocks (between 3.5 days and 7 days depending on the number of average block uncles) a new epoch will be generated. The first block of every epoch will be called its `frontier block`. 

Each time a new frontier block is executed, a new database for the `state` is created. Any trie node added during execution must be written in the last created database without exception. A suggestion is to name the new database for a certain block height as database_name(height) = height % N, but this is not mandatory. While we recommend N=20000, implementations can change this parameter, as it's internal to each node. However a low N would require retrieving state checkpoints and re-executing blocks in case of a large blockchain re-org, while a higher N reduces the space savings.


All new state data will be written to the last database created, while old data will be searched on the latest database, and if not found, on previous databases. Even if there is a blockchain reorganization, the database to start searching will be the last one created.

When switching from one epoch to another, the oldest epoch's database must be discarded. This will be done during the **epoch migration process**. There will be generally 2 active databases at the same time for states, although when a new frontier block is being executed or during the epoch migration process, which is fully asynchronous, the number of active databases can be up to 3. 

It's recommended that full nodes detect if a trie node part of the state data is not found in any database, and either notify the user and shutdown gracefully or re-start synchronizing from a known checkpoint.

**Epoch migration process**. Given two consecutive epochs `A`, `B`, their frontier blocks `a`, `b`, and databases `d(A)`, `d(B)`, a new epoch `C` will be created, with frontier block `c` and database `d(c)`. Then

1. `d(C)` is considered the new database for new updates.
1. We take `s` the `state` at `b.`
1. We travel `s` searching the trie for all entries in `d(A)` but not in `d(B)`.
1. For each entry in `d(A)` and not in `d(B)`, the entry is copied into `d(B).` Children of copied nodes could be copied automatically without checking existence in d(B).
1. `d(A)` is deleted (atomically)

**Caveat  1**. The copying of entries in d(B) in step 4 must be reflected to any other active threads before the deletion of d(A) in step 5, to avoid race conditions.

**Caveat  2**. It's possible to implement the scheme so that the garbage collector may or may not be executed on every frontier block. In this case the number of active databases could grow over 3, and lookups should follow a search order from newer to older databases. Migration however would always write to the previous to last database, and all old databases except the last two should be delated.

**Caveat  3**. The creation of a new database is assumed to be atomic (if two block processing threads are executing competing frontier blocks, only one database gets created.

**Note on Performance 1.** Performing the search in step 3 by checking first in d(A) or first in d(B) is indifferent to the result, but will affect the migration algorithm performance. If the blockchain is heavily used by a large set of users, then looking up the element in d(B) first is preferable, because this will result in only a single lookup on average. A practical heuristic would be to compare the sizes of both databases, and start with the larger one.

**Note on Performance 2.** Children of copied trie nodes could be automatically copied to d(B) without prior checking existence in d(B). These children nodes could already exist in d(B) due to trie node deduplications, but the amount of deduplications is assumed to be low. The effectiveness of this heuristic, and the cases where it could not apply should be investigated.

## Rationale

This proposal seeks to be easy to implement while improving notably the platform performance and resource consumption. As the migration method is parallelizable, it can be executed without interfering with other full node tasks and avoids database corruption. Alternative approaches have been tested on [geth](https://github.com/ethereum/go-ethereum/pull/2111) without much success. Vitalik's proposal for a mark and sweep garbage collection is requires scanning though a high number of state trees, while this construction only requires scanning a single tree.

During database migration it's possible that queries performed using the JSON-RPC interface return results, a old databases could be deleted while the JSON-RPC command is being executed. Implementations should correctly handle this edge case.


## Backwards compatibility

Once a full node has updated to support this new database scheme, the full node won't support downgrading unless a special database transformation is performed. 

## **Copyright**


Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
