```
RKSIP:
    Title: Garbage Collector
    Status: Draft
    Type: Node
    Purpose: Usability
    Author: Matías Marque<matias@rsk.co>, Sergio Demian Lerner<sergio@rsk.co>
```

# Garbage Collector for state pruning

## Abstract

Stateful blockchains generate too much information that may become useless or redundant for a node after a while. This RSKIP proposes a method to reduce storage usage without major harms.

## Motivation

Each node keeps all historical data about the states of each block generated. This data becomes more useless any time a new block is addded, as it is uncommon to access historical data and deep reorganizations become more unfeasable. This data consumes big volumes of storage while it can be regenerated if needed. This makes desirable to discard this data.

## Specification

Each block will be assigned an epoch based on its height. Every 10000 a new epoch will be generated and the first block of it will be called its `frontier block`.

Each time a new epoch is reached, a new database for the `state` is created.

For this to happen, all new state data will be written on the last database created, while old data will be searched on older databases.

There will be always 2 databases at the same time for states. This implies that when switching from one epoch to another, the oldest epoch's database must be discarded. This will be done during the epoch migration process.

**Epoch migration process**. Given two epochs `A`, `B`, their frontier blocks `a`, `b`, and databases `d(a)`, `d(b)`, a new epoch `C` will be created, with frontier block `c` and database `d(c)`. Then

1. `d(c)` is considered the new database
1. `d(b)` is copied into a new database `d´(b)`
1. We take `s` the `state` at `b`
1. We travel `s` searching all entries in `d(a)` or `d(b)`
1. For each entry in `d(a)` and not in `d(b)`, the entry is copied into `d´(b)`
1. `d(b)` is replaced by `d´(b)`
1. `d(b)`
1. `d(a)` is deleted

Caevat. If the last operation is atomic, then there's one border case to consider. If during a migration process a valid entry is not found on `d(c)`, then it is possible for that entry to not be found `d(b)` because it belongs to `d(a)`, but when it is going to be read in `d(a)` it might have already been deleted. In this case the entry must be found in `d(b)` as it was replaced `d´(b)` already.

## Rationale

This purposal seeks to be simple to implement while improving notably the platform. As this method is parallelizable and replacement of the database is amotical, it can be executed without interfering with other node tasks and avoids database corruption damage caused by node failure or similar stuff. Alternative approaches have been tested on [geth](https://github.com/ethereum/go-ethereum/pull/2111) without much success.

## Backwards compatibility

A drawback of this is the lack of backwards compatibility. Once a node is updated with this new database scheme, it will be hard to recover the old scheme.
