---
rskip: 690
title: Pegouts to multiple bitcoin address types
description: Let a pegout requester choose the type of the bitcoin address the funds are sent to
status: Draft
purpose: Usa
author: JT
layer: Core
complexity: 2
created: 07-SEP-26
---

# Pegouts to multiple bitcoin address types

## Abstract

Before RSKIP690 a pegout always ends at a legacy P2PKH address derived from the public key that
signed the rsk transaction. The requester cannot ask for anything else.

This RSKIP adds a Bridge method that takes the type of address to derive, and supports four of
them: legacy (P2PKH), segwit compatible (P2SH-P2WPKH), native segwit (P2WPKH) and taproot (P2TR).
All four are still derived from the requester's own public key.

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
P2WPKH       h                                       the same 20 bytes, different wrapper
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

Two requirements, because that string reaches the receipts trie:

1. A bech32 or bech32m address MUST be emitted in lowercase. BIP173 permits an all uppercase form,
   and a different case is a different string and therefore a different receipts root. Base58Check
   is case sensitive, so a legacy or P2SH-P2WPKH address is emitted as its encoding produces it.
2. The network dependent part of the encoding MUST come from the network, and an unrecognised
   network MUST fail rather than fall back to another one. That part is the human readable part for
   bech32 and bech32m, `bc`, `tb` and `bcrt`, and the version byte for Base58Check, `0x00` and
   `0x05` on mainnet and `0x6f` and `0xc4` on testnet and regtest. In both encodings it is covered
   by the checksum, so a wrong one does not produce a broken address. It produces a valid one that
   belongs to nobody on the chain in use.

`release_btc` carries the whole serialized bitcoin transaction. Its signature does not change, but
its content now includes output scripts that were never seen before.

### Storage

The pegout request queue is stored as a flat RLP list with three elements per queued request. Before
this RSKIP the destination is written as its 20-byte hash and nothing else, so the stored form has
never carried the address type. That works only while every entry is P2PKH.

From the activation, the destination is written as the **address string**, UTF-8 encoded. The other
two fields do not change:

```
before   RLP( hash160 (20 bytes),     amount in satoshis, rskTxHash (32 bytes),  ... )
after    RLP( address string (UTF-8), amount in satoshis, rskTxHash (32 bytes),  ... )
              └──────────────────────── one request ─────────────────────────┘
```

For a legacy request of 0.5 BTC:

```
before   f83b 94 f7ee9ab7297134a0ccc76f3d50e94def17488f2c
              84 02faf080
              a0 207052a1e6e403818fc328a3ed2e8b7e4c5a0628280c5358570c9210aa4085a0

after    f849 a2 6e3437753278564d727a6156706747444b35544a6a5a484b67674d3772384364416d
              84 02faf080
              a0 207052a1e6e403818fc328a3ed2e8b7e4c5a0628280c5358570c9210aa4085a0
```

An address is the serialization of network, type and program, so the string recovers the destination
exactly and no separate type field is needed.

**Keys.** The queue is read from more than one key, and each entry is written to exactly one of
them. That is how RSKIP146 added the `rskTxHash` without needing a migration. This RSKIP adds a
third key, `pegoutRequestQueue`, for the new format.

The key that matters here is `releaseRequestQueueWithTxHash`, since that is where every current
request lives. From the activation, entries with a `rskTxHash` are written to `pegoutRequestQueue`
instead, and `releaseRequestQueueWithTxHash` is written empty.

On read the keys are concatenated in order: `releaseRequestQueue`, `releaseRequestQueueWithTxHash`,
`pegoutRequestQueue`. That order decides which requests are batched first, so it is consensus.

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

**Why the address string in storage.** It is self describing and needs no new structure. The
alternative, a type tag next to the program, is more compact but adds a second thing to keep in sync
with the address the user was told about.

**Why migrate in one step instead of a grace period.** The Bridge already reads several queue keys
and writes each entry to exactly one of them, which is how RSKIP146 introduced the transaction hash
without a cutoff. That change could not move its old entries, because they had no transaction hash
to write. Here the entries under `releaseRequestQueueWithTxHash` already carry everything the new
format needs, so they can be rewritten immediately and no window exists where an entry is under two
keys.

## Backwards compatibility

Activation requires a hard fork.

Before activation `releaseBtcTo` does not exist and calling it fails, so no pre-fork block can queue
a non legacy destination, emit the new rejection reason, or write the new storage key. Pegouts
requested through the existing fallback keep producing a legacy address, and blocks from before the
fork replay to the same state.

Consumers that only call `releaseBtc`, and consumers of the events, keep working with the ABI they
have. A caller that wants to use `releaseBtcTo` has to add it. Separately, consumers that decode a
destination address or an output script need updating:

- Anything reading the pegout request queue from storage must handle the new key and format.
- Anything decoding `release_btc` will start seeing `a914...87`, `0014...` and `5120...` outputs.
- Anything rendering a bitcoin address must support bech32 (BIP173) and bech32m (BIP350), and must
  take the network dependent part, the human readable part or the version byte, from the network it
  is configured for.

The last one is the most dangerous. Rendering an address for the wrong network produces a valid
looking address, with a valid checksum, that belongs to nobody.

## References

[1] [RSKIP146](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP146.md): Added the rsk
transaction hash to each pegout request queue entry. It introduced the pattern this RSKIP follows
for storage: a new key alongside the existing one, read together and written to separately, with no
grace period.

[2] [RSKIP326](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP326.md): Changed
`btcDestinationAddress` in `release_request_received` from bytes to a string. That is why a bech32 or
bech32m address fits the existing event with no signature change.

[3] [BIP49](https://github.com/bitcoin/bips/blob/master/bip-0049.mediawiki): Derivation scheme for
P2WPKH nested in P2SH. It pins the redeem script to a single value, `0014 ‖ hash160(pubkey)`, which
is what makes `p2sh-segwit` derivable from a public key while generic P2SH is not.

[4] [BIP86](https://github.com/bitcoin/bips/blob/master/bip-0086.mediawiki): Key derivation for
single key P2TR outputs. Its convention, an empty merkle root, is what makes the derived output
spendable with a normal key path signature by any taproot wallet.

[5] [BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki): Segregated witness.
It defines the witness program of a P2WPKH output as `hash160(pubkey)`, the same 20 bytes as P2PKH,
which is why those two types share a derivation.

[6] [BIP173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki): Bech32, the encoding
for witness version 0 addresses. It also permits an all uppercase form, which is why this RSKIP
requires lowercase explicitly.

[7] [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki): Schnorr signatures for
secp256k1. It defines the tagged hash, the x only public key encoding and `lift_x`, the three
primitives the taproot tweak is built from.

[8] [BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki): Taproot spending rules.
It defines the tweak itself, `taproot_tweak_pubkey`, which this RSKIP applies with an empty merkle
root.

[9] [BIP350](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki): Bech32m, the encoding
for witness version 1 and later. Taproot addresses require it, and encoding one with the bech32
constant instead produces a well formed address with a wrong checksum.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
