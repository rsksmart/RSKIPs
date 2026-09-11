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

The Bridge already has everything it needs to derive the other types, since all four address types
are standard derivations from a single public key. What is missing is a way for the requester to say
which one they want, and a stored form that can represent it.

## Specification

### The new method

```
releaseBtcTo(string addressType)
```

It is payable, restricted to externally owned accounts, and the pegout amount is the value sent,
exactly like the existing `releaseBtc` fallback. Once the destination is derived, the rest is the
existing path, unchanged. `releaseBtc` keeps working and producing a legacy address.

The behavior described here is active only when `RSKIP690` is active.

`addressType` takes the names Bitcoin Core uses for its `addresstype` option:

| `addressType` | Derived type | Output script |
| --- | --- | --- |
| `legacy` | P2PKH | `76a914 <20B> 88ac` |
| `p2sh-segwit` | P2SH-P2WPKH | `a914 <20B> 87` |
| `bech32` | P2WPKH | `0014 <20B>` |
| `bech32m` | P2TR | `5120 <32B>` |

The value is matched **exactly**. No case folding, no trimming, no aliases. Any other value refunds
the amount and emits `release_request_rejected`.

The `addressType` is checked before the amount, so a call that carries both an unrecognized type and
an amount below the minimum is rejected as an unrecognized type. The reason reaches the receipts
trie, so the order is consensus.

### Address derivation

Let `P` be the compressed public key recovered from the signature of the rsk transaction, and
`h = hash160(P)`.

```
P2PKH        h
P2WPKH       h                                       the same value as P2PKH, see BIP141
P2SH-P2WPKH  hash160(0014 ‖ h)                       the inner script is pinned by BIP49
P2TR         x_only( lift_x(x_only(P)) + t·G )       where t = int(tagged_hash("TapTweak", x_only(P)))
```

The taproot output key follows BIP341 with an empty merkle root, which is the single key case
described by BIP86.

A public key for which the BIP341 tweak fails cannot produce a taproot address. The request is
refunded and rejected with `UNSUPPORTED_ADDRESS_TYPE`. This branch is not expected to be taken, but
its outcome reaches the receipts trie, so it is specified.

### Rejection reason

`release_request_rejected` gains one value:

```
UNSUPPORTED_ADDRESS_TYPE = 4
```

It must be appended. The reason travels in event data and therefore reaches the receipts trie, so
renumbering an existing value would change historical receipts on replay.

### Events

**No event signature changes.** `releaseBtcTo` emits the same `release_request_received` as
`releaseBtc`. The `btcDestinationAddress` field has been a string since RSKIP326, so a bech32 or
bech32m address goes in the existing field.

`release_btc` carries the whole serialized bitcoin transaction. Its signature does not change, but
its content now includes output scripts that were never seen before.

Two requirements on the emitted address, because it reaches the receipts trie:

1. A bech32 or bech32m address MUST be emitted in lowercase. BIP173 permits an all uppercase form,
   and a different case is a different string and therefore a different receipts root. Base58Check
   is case sensitive, so a legacy or P2SH-P2WPKH address is emitted as its encoding produces it.
2. The network dependent part of the encoding MUST come from the network, and an unrecognised
   network MUST fail rather than fall back to another one. That part is the human readable part for
   bech32 and bech32m, `bc`, `tb` and `bcrt`, and the version byte for Base58Check, `0x00` and
   `0x05` on mainnet and `0x6f` and `0xc4` on testnet and regtest. In both encodings it is covered
   by the checksum, so a wrong one does not produce a broken address. It produces a valid one that
   belongs to nobody on the chain in use.

### Storage

The pegout request queue stores the destination of each queued request. Before this RSKIP it is
written as a 20-byte hash, so the stored form cannot carry the address type. That works only while
every destination is P2PKH.

From the activation, the destination is written as its address string, UTF-8 encoded. The rest of
the entry does not change.

Requests queued before the activation are not lost and keep their position in the queue.

## Rationale

**Why these four types.** They are exactly the types Bitcoin Core can generate from a single key.
P2WSH and generic P2SH hash a script that cannot be derived from a public key, so the Bridge cannot
produce them. P2SH-P2WPKH is the exception, because BIP49 leaves exactly one valid inner script,
`0014 ‖ hash160(pubkey)`. One public key gives one script, one hash and one address, with no choice
anywhere along the way, so the Bridge can compute it.

**Why a string parameter and not an integer.** The Bridge is a precompiled contract, so parsing
happens in the node and not in EVM opcodes. The extra calldata is negligible against a pegout, and
in exchange the call is self describing and uses names the ecosystem already knows.

**Why exact matching.** The accepted set is consensus input. It can be widened later but never
narrowed, so it starts strict.

## Backwards compatibility

This change is a hard-fork and therefore all full nodes must be updated.

Before activation `releaseBtcTo` does not exist and calling it fails, so blocks from before the fork
replay to the same state.

## References

[1] [RSKIP326](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP326.md): Changed
`btcDestinationAddress` in `release_request_received` from bytes to a string. That is why a bech32 or
bech32m address fits the existing event with no signature change.

[2] [BIP49](https://github.com/bitcoin/bips/blob/master/bip-0049.mediawiki): Derivation scheme for
P2WPKH nested in P2SH. It pins the redeem script to a single value, `0014 ‖ hash160(pubkey)`, which
is what makes `p2sh-segwit` derivable from a public key while generic P2SH is not.

[3] [BIP86](https://github.com/bitcoin/bips/blob/master/bip-0086.mediawiki): Key derivation for
single key P2TR outputs. Its convention, an empty merkle root, is what makes the derived output
spendable with a normal key path signature by any taproot wallet.

[4] [BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki): Segregated witness.
It defines the witness program of a P2WPKH output as `hash160(pubkey)`, the same 20 bytes as P2PKH,
which is why those two types share a derivation.

[5] [BIP173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki): Bech32, the encoding
for witness version 0 addresses. It also permits an all uppercase form, which is why this RSKIP
requires lowercase explicitly.

[6] [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki): Schnorr signatures for
secp256k1. It defines the tagged hash, the x only public key encoding and `lift_x`, the three
primitives the taproot tweak is built from.

[7] [BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki): Taproot spending rules.
It defines the tweak itself, `taproot_tweak_pubkey`, which this RSKIP applies with an empty merkle
root.

[8] [BIP350](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki): Bech32m, the encoding
for witness version 1 and later. Taproot addresses require it, and encoding one with the bech32
constant instead produces a well formed address with a wrong checksum.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
