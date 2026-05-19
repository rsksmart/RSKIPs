---
rskip: 566
title: Bridge peg-out change & migration UTXO events
created: 06-MAY-26
author: JT
purpose: Usa
layer: Core
complexity: 1
status: Draft
description: Emit Bridge logs when peg-out change or migration UTXOs are registered via registerBtcTransaction.
---

| RSKIP | 566 |
| :---- | :----------------------- |
| **Title** | Bridge peg-out change & migration UTXO events |
| **Created** | 06-MAY-26 |
| **Author** | JT |
| **Purpose** | Usa |
| **Layer** | Core |
| **Complexity** | 1 |
| **Status** | Draft |

## Abstract

This RSKIP proposes two new Bridge contract events. They are to be emitted when Bitcoin transactions are processed through `registerBtcTransaction` and federation UTXOs are recorded from outputs that today produce **no** Bridge log: (1) peg-out **change** outputs sent back to the federation, and (2) **migration** transactions that register UTXOs on the active federation. Together they allow indexers and integrations to observe the full peg-out lifecycle and migration UTXO registration using only Bridge logs, without relying solely on off-chain heuristics.

## Motivation

The peg-out pipeline already emits Bridge events for successive stages: among others, request `release_request_received` [2], creation (`release_requested`, `batch_pegout_created`) [3], `pegout_transaction_created` [5], confirmation (`pegout_confirmed`) and signing behaviour [4], and release-related logs (`release_btc`, `add_signature`) [1][4]. When a peg-out Bitcoin transaction includes **change** paid back to the federation, that spending transaction is later processed through `registerBtcTransaction` together with other federation-facing BTC traffic. The registration path updates federation UTXO sets but does **not** emit an event, unlike a successful peg-in (`pegin_btc`). Subscribers therefore cannot close the loop from peg-out request → peg-out BTC transaction → **change UTXO registered** using only Bridge events.

Similarly, when a **migration** transaction is confirmed and registered, the same registration flow adds UTXOs without a dedicated event. Operators and indexers tracking federation balances and migration progress need an explicit, parseable signal at registration time, distinct from peg-in and peg-out change. Background on PowPeg migration flows is discussed in the broader Bridge and federation literature (for example [7]).

The Bridge already classifies incoming Bitcoin transactions from `registerBtcTransaction` into peg-in, peg-out/migration spend, or unrelated [6]. This RSKIP proposes **two** events so that, at the moment UTXOs are recorded for **peg-out change** versus **migration**, subscribers can tell those cases apart without inferring them solely from storage or Bitcoin chain inspection.

## Specification

### Overview

Introduce two events on the Bridge contract, emitted from the same precompiled Bridge address and following the same Solidity-oriented logging conventions as existing Bridge events [1]. Each event carries **only** the Bitcoin transaction identifier described below.

### `btcTxHash` parameter (both events)

For both events, `btcTxHash` is the identifier of the **Bitcoin transaction passed to `registerBtcTransaction`**—that is, the transaction whose inclusion proof is being processed on Rootstock when the new federation UTXOs are recorded.

### Event 1 — `pegout_change_output_registered`

**Semantics:** Emitted after the Bridge successfully completes processing of a Bitcoin transaction that registers UTXOs corresponding to **peg-out change** returned to the federation (per the same classification rules that treat the transaction as a Bridge-originated peg-out spend rather than a peg-in [6]).

**Signature:**

```
pegout_change_output_registered(bytes32 indexed btcTxHash)
```

### Event 2 — `migration_utxos_registered`

**Semantics:** Emitted after successful registration of UTXOs from a Bitcoin transaction classified as a **migration** transaction (funds moving from the retiring federation toward the active federation during a PowPeg composition change), distinct from ordinary peg-in or peg-out change [6].

**Signature:**

```
migration_utxos_registered(bytes32 indexed btcTxHash)
```

## Rationale

- **Observability:** Bridge logs are the standard integration surface for peg tracking; missing logs at registration force brittle off-chain inference.
- **Two events:** Peg-out change and migration differ in operational meaning; separate events avoid ambiguous decoding for subscribers.
- **Single parameter:** Only `btcTxHash` is required so subscribers can correlate this log with the registering Bitcoin transaction and derive amounts or outpoints off-chain if needed; keeping payloads minimal avoids redundancy with BTC chain data.

## Backwards Compatibility

This change updates the Bridge precompiled contract’s observable behaviour (new logs). It is a **hard fork**: all full nodes must be updated so event emission is deterministic across the network. Existing Bridge methods and storage layouts remain unchanged for callers that do not depend on logs. Indexers that subscribe to Bridge logs should be updated to consume the new events if they need migration or change-registration visibility.

## Implementations

To be completed: reference client ([rskj](https://github.com/rsksmart/rskj)) — extend `BridgeEvents`, `BridgeEventLogger` / `BridgeEventLoggerImpl`, and registration paths (e.g. `registerNewUtxos` / transaction classification per [6]); add unit and integration tests consistent with Bridge event patterns [1].

## References

[1] [RSKIP-146 — Bridge events data and formatting](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP146.md)

[2] [RSKIP-185 — Peg-out refund and events](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP185.md)

[3] [RSKIP-271 — Bridge peg-out batching](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP271.md)

[4] [RSKIP-326 — Pegout events improvements](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP326.md)

[5] [RSKIP-428 — New pegout creation event including UTXO outpoint values](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP428.md)

[6] [RSKIP-379 — Bridge peg-out and migration transactions index](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP379.md)

[7] [RSKIP-455 — PowPeg migration to multiple outputs](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP455.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
