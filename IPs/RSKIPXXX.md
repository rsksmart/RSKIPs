---
rskip: XXX
title: Peg-in v2: segwit and taproot refunds
description: Extend the RSKIP170 peg-in instructions so the Bitcoin refund address can be a P2WPKH, P2WSH or P2TR output as well as P2PKH or P2SH, and state when a version-2 refund address is used
status: Draft
purpose: Usa
author: JP
layer: Core
complexity: 2
created: 12-OCT-26
---

# Peg-in instructions version 2: segwit and taproot refund addresses

## Abstract

RSKIP170 lets a peg-in name its Rootstock destination and a Bitcoin refund address in an `OP_RETURN`
output. The refund address can only be P2PKH or P2SH. This RSKIP adds protocol version 2 of the same
payload, in which the refund address may be any of the five output types in common wallet use,
including native segwit (P2WPKH, P2WSH) and taproot (P2TR), and states the conditions under which a
version-2 refund address is used.

On the current registration path those conditions are not met by any peg-in whose payload parses,
so this RSKIP changes the ending of no peg-in by itself: it defines a deliverable refund destination
for rules introduced by later RSKIPs. Version 1 is unchanged.

## Motivation

Two things have changed since RSKIP170.

Wallets have moved to native segwit and taproot. Ledger Live, Trezor Suite and most software wallets
create `bc1q` or `bc1p` accounts by default. A user of such a wallet can peg in with a version-1
payload (the Bridge credits the destination from any input type), but cannot name a refund address of
their own type, because the version-1 field carries a 20-byte hash and a type byte that means P2PKH
or P2SH. If the peg-in is rejected, the Bridge's fallback is the address it derives from the first
input, which it can do for P2PKH, P2SH-P2WPKH, P2SH multisig and P2SH-P2WSH multisig inputs and not
for native segwit or taproot. For the wallets most people now use, a rejected peg-in has no refund
destination at all.

Protocols built on peg-in instructions need a refund destination the user chose. Intent-based
peg-ins, where the destination is a contract address derived from what the user asked for, have
endings in which the Bridge should return BTC rather than credit rBTC. Those endings need a refund
output the Bridge can pay from any input type, and a rule for when it pays. This RSKIP supplies the
destination; the rule that would give it effect, a flag under which a peg-in is credited only if its
destination has code at registration, is left to a later version so that the two can be reviewed
separately. Until such a rule is adopted, a version-2 refund address is committed by the user and
delivered by the Bridge in no case that version 1 does not already cover.

### Related work

RSKIP41 (2017, Draft) proposed a Bitcoin refund address "to transfer the remaining if no execution
occurred". RSKIP170 adopted the address for use "in case the peg-in can not be completed for some
reason" without enumerating the cases; the one case implemented, a locking-cap rejection, is checked
on the registration path that preceded RSKIP379 and not on the current one. RSKIP176 (Flyover)
carries a user refund address as a 21-byte argument with the same P2PKH-or-P2SH limitation (RSKIP284
fixed its serialisation, not its reach). RSKIP181 defined the rejection and non-refundable events
this RSKIP reuses.

RSKIP690 (Draft) lets a peg-out *requester* choose among four address types derived from their own
public key, and its implementation branches in rskj and bitcoinj-thin add the segwit and taproot
address classes and output scripts this RSKIP needs. It does not touch peg-in instructions or
refunds, and its destinations are derived from a key rather than named by the user. No RSKIP, draft
or adopted, extends peg-in refund addresses to witness programs. This RSKIP is written to build on
RSKIP690's library work rather than duplicate it.

## Specification

### Payload

A version-2 peg-in instructions output is an `OP_RETURN` output of value zero whose data is:

- `52534b54` (`RSKT`), 4 bytes;
- protocol version, 1 byte: `02`;
- Rootstock destination address, 20 bytes;
- optional refund output, encoded as:
  - script type, 1 byte;
  - program, 20 or 32 bytes according to the type.

Script types and the output script the Bridge builds from them:

| Type | Name   | Program                   | Output script                                      |
| ---- | ------ | ------------------------- | -------------------------------------------------- |
| `01` | P2PKH  | 20 bytes, public key hash | `OP_DUP OP_HASH160 <20> OP_EQUALVERIFY OP_CHECKSIG` |
| `02` | P2SH   | 20 bytes, script hash     | `OP_HASH160 <20> OP_EQUAL`                         |
| `03` | P2WPKH | 20 bytes, witness program | `OP_0 <20>`                                        |
| `04` | P2WSH  | 32 bytes, witness program | `OP_0 <32>`                                        |
| `05` | P2TR   | 32 bytes, witness program | `OP_1 <32>`                                        |

Types `01` and `02` keep their version-1 meaning. Valid data lengths are 25 (no refund), 46 (20-byte
program) and 58 (32-byte program) bytes. Any other length, type or program size is a parse failure,
handled as a version-1 parse failure is today (`PEGIN_V1_INVALID_PAYLOAD`): the payload yields no
refund output, and the Bridge refunds the sender-derived address if there is one, otherwise emits
`unrefundable_pegin`.

The payload carries the program, not a bech32 string. Address encoding is a wallet concern; the
Bridge needs only the script.

### Parsing and evaluation

`PeginInstructionsProvider` dispatches on the version byte. A new `PeginInstructionsVersion2` parses
the layout above into a Rootstock destination and an optional refund **script**. `PeginInformation`
gains a refund script alongside the existing refund address; for versions 0 and 1 the script is
derived from the address as today, so callers see one representation.

`PegUtils.evaluatePegin` currently accepts protocol versions 0 and 1 and throws on any other parsed
version. It must accept version 2 and evaluate it as version 1 (`REGISTER`), plus the refund
conditions below.

### Building the refund

In every released rskj (through VETIVER-9.0.4) refund transactions are built by
`ReleaseTransactionBuilder` to an `Address`, bitcoinj-thin's `Address` represents P2PKH and P2SH
only, and `ScriptBuilder.createOutputScript` has no witness-program form. RSKIP690's implementation
branches (rskj `add-pegout-address-type` and successors; bitcoinj-thin
`add-segwit-address-and-network-hrp`, `add-taproot-output-key-derivation`,
`dispatch-create-output-script-by-address-type`, unreleased as `0.14.4-rsk-19`) split `Address` into
`LegacyAddress` and `SegwitAddress`, add `Taproot` and `Bech32`, and build P2WPKH and P2TR output
scripts; in those branches the peg-in refund and sender addresses are typed `LegacyAddress`, so the
restriction this RSKIP lifts is made explicit rather than removed. This RSKIP requires, on top of
that work:

- P2WSH (`OP_0 <32>`) output scripts, which RSKIP690 does not need since no key derives one;
- `ReleaseTransactionBuilder` to accept any `Address` (or output script) as destination and as change
  address (`buildEmptyWalletTo` sets change to the destination);
- `PeginInformation`, `generateRejectionRelease` and its callers to carry the refund as an `Address`
  of any type rather than `LegacyAddress`.

If RSKIP690 is adopted first, this RSKIP's library work reduces to the first item.

The release transaction's inputs, signing and confirmation path are unchanged: the refund is a
rejection release as today, queued in `pegoutsWaitingForConfirmations`, signed by the federation
after the usual confirmations, and announced with `release_requested`.

### When a version-2 refund address is used

On the current registration path (RSKIP379) a peg-in whose payload parses as version 1 is always
registered, so its refund address is never consulted; only a payload that fails to parse is refunded,
and then to the sender-derived address. A version-2 refund address that was likewise never consulted
would change nothing, so the conditions are stated here.

A version-2 peg-in is refunded rather than registered when the Bridge rejects it for a reason for
which a refund is defined. The refund destination is, in order: the version-2 refund output if
present; else the sender-derived address if the first input allows one; else the peg-in is
non-refundable and `unrefundable_pegin` is emitted. The events of RSKIP181 are reused to report each
outcome; RSKIP181 defines events and reason codes, not the rejection rules themselves, which this and
later RSKIPs state.

At the time of writing no rejection with a defined refund is reachable by a parsed payload on the
current path (the locking-cap refund of the pre-RSKIP379 path is the only precedent), so no peg-in
accepted today ends differently under this RSKIP. A later RSKIP that defines a refund for a version-2
payload uses this destination without change to the payload; a later rule that needs new data from
the user, such as a flag under which a peg-in is credited only if its destination has code at
registration, requires a version 3 layout, since the version-2 lengths are fixed here.

The minimum peg-in value applies to each output paying a live federation. A peg-in with any such
output below the minimum is non-refundable in every version, since the Bridge does not build
releases for it.

### Events

No new events. A refund to a version-2 output emits `rejected_pegin` and `release_requested` as a
version-1 refund does. `unrefundable_pegin` with reason `PEGIN_V1_REFUND_ADDRESS_NOT_SET` keeps its
number and meaning: no refund output in the payload and no sender-derived address either.

## Rationale

**A type table rather than a raw script.** Carrying the scriptPubKey itself would cover future
witness versions without a new RSKIP, and could be checked against the same five templates. The
table is preferred because it is shorter on the wire, puts the template check in the parser where
every other field is checked, and leaves no encoding in which a non-standard script is even
expressible. The Bridge must not pay a non-standard output: nodes on default policy do not relay
such a transaction, and a signed refund that does not propagate leaves the funds in the release
pipeline.

**Keeping the refund optional.** A version-2 payload without a refund output behaves exactly as
version 1. This lets the version byte signal the layout without forcing wallets that have no refund
address to invent one.

**Not reusing the version-1 type byte.** Extending version 1 with types `03` to `05` would be
shorter, but the 32-byte programs change the valid lengths, and the Bridge's version-1 parser
hard-codes 25 and 46 (`PeginInstructionsVersion1.validateDataLength`), as do tools that read the
payload. A new version number keeps every existing parser correct by construction.

**Stating the refund conditions.** RSKIP170 named the purpose of the refund address and left the
registration path free to ignore it, which the current path does. A field that is committed by the
user and never read suggests a protection that does not exist. The conditions here are deliberately
conservative, so that this RSKIP's effect on existing peg-ins is nil, and explicit, so that later
rules have a destination to use.

## Backwards compatibility

This change is a hard fork and all full nodes must be updated.

Version 1 payloads are untouched. Nodes before activation reject a version-2 payload as an invalid
version, which under RSKIP379 means a refund to the sender-derived address or `unrefundable_pegin`;
so wallets must not emit version 2 before the activation height. After activation, a version-2
payload without a refund output is indistinguishable in effect from version 1.

`PeginInformation`'s stored form does not change; refund scripts are not persisted, the release
transaction is.

## Security considerations

- The refund output is chosen by whoever builds the `OP_RETURN`. For a user signing their own
  transaction this is their wallet; for a protocol building the payload on the user's behalf, the
  user's software must verify it, as it must verify the destination. This RSKIP does not change that
  trust boundary.
- Refund size. The refund output is the deposit less the release fee (`ReleaseTransactionBuilder`
  builds with `recipientsPayFees` and `emptyWallet`). Whether the net output clears the dust
  threshold of its type depends on the deposit, the release transaction's size and the fee-per-kb
  the federation has voted, which on mainnet may be set as high as 0.05 BTC per kB
  (`FeePerKbMainNetConstants.maxFeePerKb`); at that ceiling a minimum-value deposit does not cover a
  one-input release. Where the net output is dust or cannot be adjusted, the builder returns
  `DUSTY_SEND_REQUESTED` or `COULD_NOT_ADJUST_DOWNWARDS` and the Bridge logs and panics, exactly as
  for a version-1 refund today. This RSKIP neither adds nor removes that failure.
- The powHSM needs no change. Its signing authorisation (`auth_tx.c`, `auth_receipt.c`, protocol v5)
  hashes the transaction's outputs as bytes into the transaction hash and the sighash and never
  interprets them; it authorises a signature when a Bridge `release_requested` log in a proved
  receipt names that transaction hash. A refund to a witness-program output is signed exactly as a
  P2PKH refund is, provided the Bridge emits `release_requested` for it, which the rejection-release
  path does.
- A 32-byte program lengthens the `OP_RETURN` data to 58 bytes. Bitcoin Core 30.0 raised the default
  data-carrier limit to 100,000 bytes; 58 bytes also fits the 80-byte default of earlier versions,
  which operators can still enforce (`-datacarriersize=83`).

## Test cases

- Parse: each type with its correct program length; wrong length for a type; unknown type; a 58-byte
  payload with a 20-byte type (rejected).
- Register: a version-2 payload from P2PKH, P2WPKH and P2TR inputs, every federation output at or
  above the minimum, is credited to the destination exactly as version 1 (no behavioural change on
  acceptance).
- Minimum: a version-2 payload with one federation output below the minimum is non-refundable, as in
  version 1.
- Refund: with a refundable rejection reason injected (test-only until a rule exists), the release
  pays an output of the declared script with the deposit less fee; for each of the five types the
  output is accepted by Bitcoin Core as standard (`testmempoolaccept`).
- Fallback: a refundable rejection of a version-2 payload with no refund output from a P2PKH input
  refunds the sender-derived address; from a P2WPKH input emits `unrefundable_pegin` with
  `PEGIN_V1_REFUND_ADDRESS_NOT_SET`.
- Pre-activation: a version-2 payload is treated as an invalid version.
- Regression: all RSKIP170 vectors unchanged.

## Open questions

- Whether to include the flags byte now (bit 0: credit only if the destination has code, else
  refund), which would give this RSKIP an immediate effect, or keep it for a version 3 as drafted
  here.
- Sequencing with RSKIP690, which gives peg-outs key-derived segwit and taproot destinations and
  carries the library changes; whether this RSKIP should wait for it, or the two should share one
  network upgrade. A user-named peg-out destination (as opposed to a key-derived one) remains
  outside both.
- Whether RSKIP176's user refund address should accept the same table, for Flyover refunds on
  locking-cap rejections.

## References

[1] [RSKIP170](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP170.md): Peg-in to any
address

[2] [RSKIP41](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP41.md): Extended Bitcoin
Bridge transactions (Draft)

[3] [RSKIP176](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP176.md): Programmable peg-in
addresses; [RSKIP284](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP284.md): Flyover
refund address serialisation

[4] [RSKIP181](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP181.md): Peg-in rejection
events

[5] [RSKIP379](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP379.md): Peg-out and
migration transactions index, the current registration path

[6] [RSKIP690](https://github.com/rsksmart/RSKIPs/pull/690): Pegouts to different bitcoin address
types (Draft); rskj branch `add-pegout-address-type`, bitcoinj-thin branches
`add-segwit-address-and-network-hrp` and `add-taproot-output-key-derivation`, checked 12-OCT-26

[7] rskj VETIVER-9.0.4: `PeginInstructionsVersion1`, `PegUtils.evaluatePegin` and
`allUTXOsToFedAreAboveMinimumPeginValue`, `BtcLockSenderProvider` and `P2shP2wshBtcLockSender`,
`BridgeSupport.generateRejectionReleaseFromFederation`, `ReleaseTransactionBuilder`,
`FeePerKbMainNetConstants`; bitcoinj-thin `Address`, `ScriptBuilder`

[8] [rsk-powhsm](https://github.com/rsksmart/rsk-powhsm): `firmware/src/powhsm/src/auth_tx.c`,
`auth_receipt.c`; `docs/protocol.md` (v5)

[9] [Bitcoin Core 30.0 release notes](https://github.com/bitcoin/bitcoin/blob/master/doc/release-notes/release-notes-30.0.md):
`-datacarriersize` default

[10] [BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki): Segregated witness;
[BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki): Taproot

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
