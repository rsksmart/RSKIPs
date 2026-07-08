---
rskip: 378
title: Enforce release transaction size limit
created: 06-JUL-26
author: JZ
purpose: Usa, Sec
layer: Core
complexity: 1
status: Draft
description: Adds a safety margin for the size (SegWit virtual size, following BIP 141) of the release transactions the Bridge creates (peg-outs and migrations).
---

|RSKIP          |378           |
| :------------ |:-------------|
|**Title**      |Enforce release transaction size limit |
|**Created**    |06-JUL-26 |
|**Author**     |JZ |
|**Purpose**    |Usa, Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP adds a safety margin below Bitcoin's standard transaction size limit for the release transactions the Bridge creates (peg-outs and migrations after a federation change), keeping them conservatively within the size that Bitcoin nodes will relay. During construction, the Bridge measures a transaction's SegWit virtual size (vsize) following BIP 141 and rejects it if the vsize exceeds the standard limit reduced by the margin. The behavior is gated by this RSKIP's activation and takes effect only from the corresponding network upgrade onwards.

## Motivation

Release transactions (peg-outs and migrations) can accumulate many inputs and outputs, so their size must stay under Bitcoin's standard transaction size limit; otherwise Bitcoin nodes will not relay them and the peg-out/migration flow stalls.

To keep these transactions safely within that limit — with headroom to spare — this RSKIP reserves a safety margin below the standard size limit and checks every release transaction against the reduced bound before finalizing it, rejecting the build if it does not fit. The size used for the check is the transaction's virtual size, which for a SegWit federation accounts for the witness discount so it reflects the real size Bitcoin nodes measure. Because this value is computed during transaction construction, it must be reproduced identically by every node validating the chain and is therefore a consensus rule.

## Specification

The behavior described here is active only when `RSKIP378` is active (see *Backwards Compatibility*).

A safety margin of 10,000 vbytes is reserved below Bitcoin's standard transaction size limit of 100,000 vbytes, resulting in a 90,000 vbytes bound. Release transactions are checked against it: during construction, a release transaction whose estimated size exceeds this bound is **rejected**.

The size is measured as the transaction's **virtual size (vsize)** as defined in BIP 141, and it is estimated over the unsigned transaction.

## Rationale

- **Conservative safety margin.** The check enforces a 90,000-vbyte bound — 10,000 vbytes below Bitcoin's 100,000-vbyte standard limit — rather than the exact limit. This margin, together with upper-bound estimation, keeps every release transaction comfortably within the standard size and guarantees it will be relayed even under conservative assumptions.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated.

## References

[1] BIP 141 — Segregated Witness (Consensus layer): transaction weight and virtual size. https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki

[2] BIP 144 — Segregated Witness (peer services): witness serialization. https://github.com/bitcoin/bips/blob/master/bip-0144.mediawiki

[3] RSKIP305 — Peg-out efficiency improvement (Segwit): introduces the SegWit (P2SH-P2WSH) federation. https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md

[4] bitcoinj-thin — Bitcoin standard transaction size limit, `BtcTransaction.MAX_STANDARD_TX_SIZE` (`co.rsk.bitcoinj.core`). https://github.com/rsksmart/bitcoinj-thin

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
