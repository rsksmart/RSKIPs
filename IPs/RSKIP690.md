---
rskip: 690
title: Pegouts to different bitcoin address types
description: Let a pegout requester choose the type of the bitcoin address the funds are sent to
status: Draft
purpose: Usa
author: JT
layer: Core
complexity: 2
created: 07-SEP-26
---

# Pegouts to different bitcoin address types

## Abstract

Before RSKIP690 a pegout always ends at a legacy P2PKH address derived from the public key that
signed the rsk transaction sending funds to the Bridge. The requester cannot ask for anything else.

This RSKIP adds a Bridge method that takes the type of address to derive, and supports the four
types that can be derived from the requester's own public key: legacy (P2PKH), segwit compatible
(P2SH-P2WPKH), native segwit (P2WPKH) and taproot (P2TR).

## Motivation

Most of the Bitcoin network has moved to segwit. Users who hold and spend from segwit or taproot
wallets receive their pegouts at a legacy address they may not even use, and they pay more to spend
that output than they would from a segwit one.

Since all four address types are standard derivations from a single public key, the user should be
able to say which one they want for the Bridge to derive it.

## Specification

### The new method

```solidity
function releaseBtcTo(string calldata addressType) external payable;
```

It is restricted to externally owned accounts, and the pegout amount is the value sent, exactly
like the existing `releaseBtc` fallback. Once the destination is derived, the rest is the
existing path, unchanged. `releaseBtc` keeps working and producing a legacy address.

The behavior described here is active only when `RSKIP690` is active.

`addressType` takes the names Bitcoin Core uses for its `addresstype` option:

| `addressType` | Derived type | Output script |
| --- | --- | --- |
| `legacy` | P2PKH | `76a914 <20B> 88ac` |
| `p2sh-segwit` | P2SH-P2WPKH | `a914 <20B> 87` |
| `bech32` | P2WPKH | `0014 <20B>` |
| `bech32m` | P2TR | `5120 <32B>` |

The value is matched **exactly**. No case folding, no trimming, no aliases.

A call can fail more than one check. Exactly one `release_request_rejected` is emitted, carrying
the reason of the first failure in this order:

1. Caller is a contract. `CALLER_CONTRACT`, not refunded.
2. `addressType` is not one of the four. `UNSUPPORTED_ADDRESS_TYPE`, refunded.
3. The amount fails the existing checks. `LOW_AMOUNT` or `FEE_ABOVE_VALUE`, refunded.
4. The taproot tweak fails. `UNSUPPORTED_ADDRESS_TYPE`, refunded.

### Address derivation

Let `P` be the compressed public key recovered from the signature of the rsk transaction,
`h = hash160(P)` and `t = int(tagged_hash("TapTweak", x_only(P)))`. `tagged_hash`, `x_only` and
`lift_x` are defined in BIP340.

```
P2PKH        h
P2WPKH       h
P2SH-P2WPKH  hash160(0014 ‖ h)
P2TR         x_only( lift_x(x_only(P)) + t·G )
```

The taproot output key follows BIP341 with an empty merkle root, which is the single key case
described by BIP86.

The BIP341 tweak can fail, and a public key for which it fails cannot produce a taproot address.
That is the fourth case in the rejection order above.

### Events

**No event signature changes.** `releaseBtcTo` emits the same `release_request_received` as
`releaseBtc`. The `btcDestinationAddress` field has been a string since RSKIP326, so a bech32 or
bech32m address goes in the existing field.

`release_btc` carries the whole serialized bitcoin transaction. Its signature does not change, but
its content now includes output scripts that were never seen before.

`release_request_rejected` gains one reason value, appended to the ones RSKIP185 assigns:

- **4**: the address type is not supported.

Value **3** is already taken by the fee above value rejection, which no RSKIP documents.

Two requirements on the emitted address, because it reaches the receipts trie:

1. A bech32 or bech32m address MUST be emitted in lowercase. Base58Check is case sensitive, so a
   legacy or P2SH-P2WPKH address is emitted as its encoding produces it.
2. The human readable part of a bech32 or bech32m address, and the version byte of a base58
   address, MUST come from the network. There is no default.

### Storage

The pegout request queue stores the destination of each queued request. Before this RSKIP it is
written as a 20-byte hash, so the stored form cannot carry the address type. That works only while
every destination is P2PKH.

From the activation, the destination is written as its address string, UTF-8 encoded. The rest of
the entry does not change.

Requests queued before the activation are not lost and keep their position in the queue.

## Rationale

**Why these four types.** They are the ones that derive from a single public key. P2WSH and generic
P2SH hash a script, and a public key does not determine a script. P2SH-P2WPKH is derivable only
because BIP49 leaves exactly one valid inner script.

**Why a string and not an integer.** The Bridge is a precompiled contract, so parsing costs no EVM
opcodes, and in exchange the call is self describing and uses names the ecosystem already knows.

**Why exact matching.** The accepted set is consensus input. It can be widened later but never
narrowed.

**Why the address is pinned to lowercase and to the network.** It reaches the receipts trie, so a
different string is a different receipts root. A wrong human readable part does not produce an
invalid address, it produces a valid one for another network.

**Why the rejection order is fixed.** A call can fail more than one check and has to give the same
reason on every node. The contract check comes first because it is the only one that does not
refund; any later and a contract could get its value back by passing an unknown `addressType`.

## Backwards compatibility

This change is a hard-fork and therefore all full nodes must be updated.

Before activation `releaseBtcTo` does not exist and calling it fails, so blocks from before the fork
replay to the same state.

## References

[1] [RSKIP185](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP185.md): Peg-out refund and
events, which assigns the existing `release_request_rejected` reasons

[2] [RSKIP326](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP326.md): `btcDestinationAddress`
changed from bytes to a string

[3] [BIP49](https://github.com/bitcoin/bips/blob/master/bip-0049.mediawiki): Derivation scheme for
P2WPKH nested in P2SH, which pins the redeem script to a single value

[4] [BIP86](https://github.com/bitcoin/bips/blob/master/bip-0086.mediawiki): Key derivation for
single key P2TR outputs, the empty merkle root convention

[5] [BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki): Segregated witness,
which defines the P2WPKH witness program as the same 20 bytes as P2PKH

[6] [BIP173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki): Bech32, the encoding
for witness version 0 addresses, which also permits an all uppercase form

[7] [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki): Schnorr signatures,
which define the tagged hash, the x only encoding and `lift_x`

[8] [BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki): Taproot spending
rules, which define the tweak this RSKIP applies with an empty merkle root

[9] [BIP350](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki): Bech32m, the encoding
required for witness version 1 and later

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
