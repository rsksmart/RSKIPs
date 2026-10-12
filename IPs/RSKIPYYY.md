---
rskip: YYY
title: Peg-in v3: conditional credit, else refund
description: Add a flags byte to the peg-in instructions; with the first flag set, the Bridge credits the destination only if it has code when the peg-in is registered, and otherwise refunds the deposit to the refund output
status: Draft
purpose: Usa
author: JP
layer: Core
complexity: 2
requires: 704
created: 12-OCT-26
---

# Peg-in instructions version 3: conditional credit, else refund

## Abstract

Peg-in instructions version 2 (RSKIP704) let a peg-in name a refund output of any common script
type, but define no case in which a parsed payload is refunded. This RSKIP adds protocol version 3,
which carries a flags byte. With the `REQUIRE_CODE` flag set, the Bridge credits the destination
only if that address holds contract code at the moment the peg-in is registered; if it holds none,
the Bridge refunds the deposit to the refund output instead. A version-3 payload with the flag clear
behaves as version 2.

This is the first rule that reads the refund output. It lets a Bitcoin payment be conditional on
something having happened on Rootstock, without the Bridge knowing what, and it gives a user whose
peg-in was meant to trigger a contract a way to get their BTC back when nothing was triggered.

## Motivation

A peg-in can name a contract as its destination (RSKIP170) and the Bridge credits it whether or not
anything is there. Protocols that derive the destination from the user's intent, so that the
address itself commits to what should happen, have a failure ending the Bridge cannot see: nobody
acted on the intent, the rBTC is credited to an address with no code, and the user's BTC is now rBTC
in a contract they may be unable to spend from, since they hold no Rootstock key and no rBTC for
gas. Returning it needs either a Rootstock transaction the user cannot send, or a third party paid
to send BTC on their behalf.

The Bridge can tell whether an address has code at registration, roughly a hundred Bitcoin blocks
after payment. In a protocol where code is placed at the destination only when the intent has been
acted on (for example, by a factory that deploys there on execution), "has code" is a faithful
signal of "was acted on", and the Bridge can refund without knowing the protocol. RSKIP41 (2017,
Draft) proposed exactly this ending, a Bitcoin refund "to transfer the remaining if no execution
occurred"; RSKIP170 adopted the destination and refund fields and left the condition out.

The flag is opt-in. A peg-in without it, or with an older payload version, is credited as today.

## Specification

### Payload

A version-3 peg-in instructions output is an `OP_RETURN` output of value zero whose data is:

- `52534b54` (`RSKT`), 4 bytes;
- protocol version, 1 byte: `03`;
- Rootstock destination address, 20 bytes;
- flags, 1 byte;
- optional refund output, encoded as in version 2 (RSKIP704): script type, 1 byte; program, 20 or
  32 bytes.

Valid data lengths are 26 (no refund), 47 (20-byte program) and 59 (32-byte program) bytes.

Flags:

| Bit | Name           | Meaning                                                                 |
| --- | -------------- | ----------------------------------------------------------------------- |
| 0   | `REQUIRE_CODE` | Credit only if the destination has code at registration; else refund   |
| 1–7 | reserved       | Must be zero                                                            |

A payload with any reserved bit set is a parse failure. A payload with `REQUIRE_CODE` set and no
refund output is a parse failure, so that no peg-in can commit to a refund it cannot receive. Parse
failures are handled as in version 2: the sender-derived address is refunded if one exists,
otherwise `unrefundable_pegin` is emitted.

### Evaluation

`PegUtils.evaluatePegin` evaluates a parsed version-3 payload as follows, after the existing
minimum-value check and before anything else:

1. If `REQUIRE_CODE` is clear: `REGISTER`, as version 2.
2. If `REQUIRE_CODE` is set and the destination has non-empty code in the Rootstock state at the
   block in which `registerBtcTransaction` executes: `REGISTER`.
3. Otherwise: `REFUND`, with the new rejection reason `PEGIN_DESTINATION_HAS_NO_CODE`.

"Has code" means the account's code is non-empty. A precompiled contract address counts as having
code. The check is made against the state of the Rootstock block that includes the registration,
not against any Bitcoin block height.

The refund follows the version-2 rule: it is built to the refund output, which is mandatory when the
flag is set, by the existing rejection-release path (`generateRejectionRelease`), for the deposit
less the release fee, from the deposit's own UTXOs. The transaction is marked processed.

### Events

`rejected_pegin` (RSKIP181) gains one reason value, appended to the existing ones:

- `PEGIN_DESTINATION_HAS_NO_CODE`: the payload required code at the destination and found none.

The refund emits `release_requested` as every rejection release does. No event signature changes.

## Rationale

**Why a flag rather than a new rule for every peg-in.** Crediting an address without code is the
behaviour RSKIP170 chose on purpose, so that funds can be sent ahead of deployment. The flag keeps
that default and lets a payment opt out of it.

**Why "has code" and not anything more specific.** Any richer condition (a method the Bridge calls,
a registry it consults) would make the Bridge depend on a particular contract's interface, and
consensus code should not. Code presence is a property of the state the Bridge already holds, costs
one lookup, and is enough for any protocol that can arrange to deploy on success: with CREATE2 the
address is known in advance, and only the designated deployer can put code there.

**Why the refund output is mandatory with the flag.** A refund the Bridge cannot deliver is funds
stranded in the federation's wallet with no record. Requiring the output at parse time means the
user's own wallet rejects the combination before any BTC moves.

**Why the check is at registration.** It is the only moment the Bridge acts on a peg-in. A
consequence for protocols: code must be in place before the PowPeg registers, and a deployment
that lands after registration is too late, since the refund is already queued. Registration
happens about a hundred Bitcoin blocks after the payment, so a protocol that acts within hours has
ample margin; the residual race is theirs to price.

**Why reserved bits must be zero.** A later version can assign them without a parser on any node
having silently accepted a payload whose meaning it did not know.

## Backwards compatibility

This change is a hard fork and all full nodes must be updated.

Versions 0, 1 and 2 are untouched. Nodes before activation reject a version-3 payload as an invalid
version, which under RSKIP379 means a refund to the sender-derived address or `unrefundable_pegin`;
so wallets must not emit version 3 before the activation height.

The new rejection reason is appended, so existing consumers of `rejected_pegin` see a value they do
not recognise only for version-3 peg-ins.

## Security considerations

- **Forcing a credit.** An attacker who wanted a flagged peg-in credited rather than refunded would
  need to place code at the destination. For a CREATE2 address only the designated deployer can; for
  an address nobody can deploy to, the peg-in is always refunded, which is the user's own choice.
- **Forcing a refund.** Code cannot be removed from an address by a third party, so nobody can turn
  an intended credit into a refund. And a refund pays the user's own refund output, so even a
  refund the user did not want is a return of their funds, less the release fee.
- **Refund address substitution.** The refund output is chosen by whoever builds the `OP_RETURN`, as
  in version 2; a user who does not build their own payload must verify it, as they must verify the
  destination. Unchanged by this RSKIP.
- **Refund size and fees.** As in version 2: the net output can be dust or unbuildable at extreme
  fee-per-kb settings, in which case the builder refuses and the Bridge logs and panics, as for any
  rejection release today.
- **Cost to the federation.** Each refund is a federation-signed Bitcoin transaction paid from the
  deposit. A protocol whose intents are often unserved generates refunds at its users' expense, not
  the federation's.
- **powHSM.** No change: the device authorises a release by matching the transaction hash to a
  `release_requested` log and does not interpret outputs (RSKIP704, security considerations).

## Test cases

- Parse: 26, 47 and 59-byte payloads with flags `00` and `01`; a reserved bit set (rejected); flag
  set with no refund output (rejected); each refund type from version 2.
- Credit: a version-3 peg-in with the flag set to a destination that has code is credited exactly as
  version 2.
- Refund: a version-3 peg-in with the flag set to a destination without code emits `rejected_pegin`
  with `PEGIN_DESTINATION_HAS_NO_CODE` and `release_requested`, the release pays the refund output
  for the deposit less fee, and the transaction is marked processed; repeated for each of the five
  refund types, each output accepted by Bitcoin Core as standard.
- Timing: code deployed in the same Rootstock block before the registration transaction leads to a
  credit; deployed in the block after leads to a refund.
- Flag clear: a version-3 peg-in with flags `00` to a code-less destination is credited.
- Precompile destination: credited.
- Pre-activation: a version-3 payload is treated as an invalid version.
- Regression: all version-1 and version-2 vectors unchanged.

## Open questions

- Whether a delegated account under a future account-abstraction scheme should count as having
  code. Not reachable at the time of writing.
- Whether the check should accept a destination that had code at any earlier block, rather than at
  the registration block, to remove the deployment race. It would need a record the Bridge does not
  keep today, so this draft does not propose it.

## References

[1] [RSKIP704](https://github.com/rsksmart/RSKIPs/pull/704): Peg-in instructions version 2, segwit
and taproot refund addresses (Draft)

[2] [RSKIP170](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP170.md): Peg-in to any
address

[3] [RSKIP41](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP41.md): Extended Bitcoin
Bridge transactions (Draft), the refund "if no execution occurred"

[4] [RSKIP181](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP181.md): Peg-in rejection
events

[5] [RSKIP379](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP379.md): Peg-out and
migration transactions index, the current registration path

[6] rskj VETIVER-9.0.4: `PegUtils.evaluatePegin`, `BridgeSupport.registerBtcTransaction`,
`BridgeSupport.generateRejectionRelease`, `Repository.isContract`

[7] [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014): CREATE2, the deployment scheme under which
only a designated deployer can place code at a pre-known address

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
