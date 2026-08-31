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

`getNextPegoutWithEnoughConfirmations` returns the first match found while iterating a `HashSet`,
whose iteration order is an internal detail of the JDK. The pegout it selects can therefore differ
between JDK versions. This RSKIP replaces that rule with an explicit ordering.

## Motivation

A consensus-relevant selection must not depend on the JDK. The current rule binds nodes to a
specific set of JDK versions, and it is impractical to reimplement in another language.

## Specification

From the activation of this RSKIP, `getNextPegoutWithEnoughConfirmations` returns the entry with
enough confirmations that is the minimum under `BTC_TX_COMPARATOR`, which orders entries by the
lexicographical byte order of the serialized btc transaction.

## Rationale

`BTC_TX_COMPARATOR` already exists, orders any two entries, and depends only on data that is already
part of consensus. The resulting rule is a single sentence and can be reimplemented anywhere.

## Backwards compatibility

Activation requires a hard fork.

Blocks from before the fork must keep replaying to the same state on any JDK. Every historical call
that had more than one eligible entry is therefore hardcoded in the node, as a per-network map from
the hash of the `updateCollections` rsk transaction to the hash of the selected btc transaction.
Calls with zero or one eligible entry were already deterministic and are not recorded.

Before activation the node looks the call up in that map, and on a miss keeps the original selection.
The comparator is not used before activation: it would change pre-fork results that are not in the
map, and so diverge from nodes that have not upgraded yet.
