---
rskip: 559
title: Deterministic next pegout selection fix
description: Introduce a clear solution for selecting the next pegout with enough confirmations
status: Draft
purpose: Sec
author: VZ
layer: Core
complexity: 2
created: 16-APR-26
---

# Deterministic next pegout selection fix

## Abstract

It was determined that the output of `getNextPegoutWithEnoughConfirmations` is almost non-deterministic.
We return the first valid match from a collection whose iteration order depends on the JDK's internal implementation.
This is why we had issues when switching to Java 21.

Such behavior is very hard to predict and it effectively binds nodes to a specific set of JDK versions.
Replicating this behavior in any other programming language is extremely hard.

Elements ordering algorithm must be clear and easy to understand.


## Specification & Rationale

The old selection rule is barely describable because it depends on the ordering of the elements in the `HashSet`s, where one is fed into another.

The new selection rule utilizes the existing `BTC_TX_COMPARATOR`,
which uses lexicographical byte ordering of serialized transactions.
`(input collection of unknown order)->[BTC_TX_COMPARATOR]->find first match`

This approach is easy, clean, and understandable.

Eventually the old selection code and Java 17+ compatibility fixes must be removed.


## Backwards compatibility

Changes require a hard fork that turns on deterministic element ordering via `BTC_TX_COMPARATOR`.

After the hard fork, all conflicting historical outputs of `getNextPegoutWithEnoughConfirmations`
should be extracted and hardcoded into the node.
This will allow us to completely replace the Java-dependent code.

So the final logic flow for `getNextPegoutWithEnoughConfirmations` will be as follows:

 - call the function to get the hardcoded pegout and return it if it exists
 - if no pegouts are found, fall back to selecting a new pegout from the collection sorted with `BTC_TX_COMPARATOR`

No logic branching is required - hardcoded pegouts exist only before the hard fork,
this covers all cases where the new selection logic differs from the historical one.

It is not necessary, but it is plausible, to hardcode all historical outputs for cases where there was more than one entry to select from.

## Testnet v3 additional logic

`PegoutsWaitingForConfirmation` storage is mutable,
it updates after processing of the output taken from `getNextPegoutWithEnoughConfirmations` is finished.
On the testnet there are several blocks where the next pegout is requested multiple times per block.

Considering this additional logic is required to search for the hardcoded pegout.
Also, for historical blocks, a sequence of pegouts should be saved for such cases.

