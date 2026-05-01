---
rskip: 559
title: Deterministic next pegout selection fix
description: Intorduce clear solution to select next pegout with enough confirmations
status: Draft
purpose: Sec
author: VZ
layer: Core
complexity: 2
created: 2026-04-16
---

## Abstract

It was determined that we have almost non deterministic behaviour for output of the function `getNextPegoutWithEnoughConfirmations`.
We are returning first valid match from collection which ordering is depends on JDK internal implementation.
This is why we were having issues switching to Java 21.

Such behaviour is very hard to predict and it bins nodes to exact set of JDKs.
Replicating such behaviour in any other programming language is extremely hard.

Elements ordering algorithm must be clear and easy to understand.


## Specification & Rationale

Old ordering rule is barely describable becuase it depends on ordering in 2 HashSets where one is feeded into another.

New ordering just reuse existing `BTC_TX_COMPARATOR` that just use lexicographical bytes ordering of serialized transactions.
This approach is easy, clean and understandable.


## Backwards compatibility

Changes reqires a hardfork that will turn on valid elements ordering.
After hardfork it will be possible to extract all conflicting outputs of `getNextPegoutWithEnoughConfirmations` and hardcode them in mainnet node configuration.
After that old ordering code will be removed.

For other types of networks like future testnet v4 etc it will be required to activate this RSKIP from the block one. Testnet 3 will be morelikely dismissed till that time so no need to hardcode any outputs for it.
