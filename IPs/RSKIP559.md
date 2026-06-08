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

Old ordering rule is barely describable because it depends on the iteration order of two `HashSet`s, where one is fed into another.

The new ordering reuses the existing `BTC_TX_COMPARATOR`, which uses lexicographical byte ordering of serialized transactions.
This approach is easy, clean, and understandable.


## Backwards compatibility

Changes require a hard fork that turns on deterministic element ordering.
After the hard fork, it will be possible to extract all conflicting outputs of `getNextPegoutWithEnoughConfirmations` and hardcode them in the Mainnet node configuration.
After that, the old ordering code can be removed.

For other networks (e.g., a future Testnet v4), it will be required to activate this RSKIP from block 1.
### Testnet 3 corner case

`PegoutsWaitingForConfirmation` storage is mutable, it changes state on `getNextPegoutWithEnoughConfirmations`.
Which leads to complex behaviour if there are multiple TX calling bridge contract in the same block.
In the testnet 3 we have such collisions on blocks 2589068 and 2589112.
So it is not possible to hardcode pegouts for a testnet 3 using simple key-value map.

Considering that testnet 3 is near EOL it is unreasonable to introduce additional complexity
with state mutation for historical pegouts while it is not required for the mainnet.
I suggest to do not harcode conflicting pegouts and leave old sorting code until testnet 3 will be dismissed.

