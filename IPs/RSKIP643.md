---
rskip: 643
title: Registering peg-out and migration transactions by hash
created: 29-SEP-26
author: JT
purpose: Sca
layer: Core
complexity: 2
status: Draft
description: Let a peg-out or migration transaction be registered by its hash, without resubmitting the transaction the Bridge itself created
---

# Registering peg-out and migration transactions by hash

## Abstract

Registering a Bitcoin transaction in the Bridge requires submitting the whole serialized
transaction to `registerBtcTransaction`. For a peg-out or a migration this is redundant: the Bridge
built that transaction, so it already knows its outputs.

This RSKIP adds a Bridge method that takes the transaction hash and a proof that it has enough
Bitcoin confirmations. When the Bridge builds the transaction it records, under that hash, the
outputs that pay back to a federation, and credits them when the proof arrives.

## Motivation

None of the bytes of a release transaction tell the Bridge anything it does not already have. It
picked the inputs, built the outputs, and held the transaction while it waited for signatures. The
one thing it cannot know is whether that transaction reached Bitcoin and is buried deep enough,
which is what the proof is for.

Sending them anyway ties the cost of a registration to the shape of the transaction being
registered. Sending the hash does not: a registration is the same size whatever the release
transaction turned out to be.

## Specification

### The new method

```solidity
function registerPegoutTransaction(bytes32 btcTxHash, int height, bytes calldata pmt) external;
```

The method exists only once `RSKIP643` is active, and anyone may call it. `height` and `pmt` mean
what they mean in `registerBtcTransaction` and have the same types, so a caller that can build a
proof for one can build it for the other.

`btcTxHash` is the transaction hash without the witness. The call is rejected, with no state change,
when any of the following holds:

1. The hash has already been registered.
2. The height does not have enough confirmations, or the partial merkle tree does not prove the hash
   at that height.
3. The hash is not in the index described below.

Otherwise the Bridge credits the outputs recorded for that hash to the federation each one pays to,
removes the entry from the index, and marks the hash as registered.

### The index

When the Bridge creates a release transaction it records, under that transaction's hash, the outputs
of that transaction that pay to the active or the retiring federation. Outputs paying anywhere else
are not recorded, since the federation never receives them.

An entry is never empty. A release transaction that pays nothing back to a federation is not
recorded, and its hash is therefore rejected like any other hash the index does not hold.

This index is how the Bridge now recognises its own release transactions, in place of the peg-out
transaction index of RSKIP379. That one is still read during the transition described below.

### Events

Two events are added. Both are emitted whenever outputs are credited to a federation, whichever
method did it.

```solidity
event utxos_registered(bytes32 indexed btcTxHash, bytes values, bytes outputIndexes, string federationBtcAddress);
event flyover_utxos_registered(bytes32 indexed btcTxHash, bytes values, bytes outputIndexes, string federationBtcAddress, bytes32 flyoverDerivationHash);
```

`values` and `outputIndexes` are the amounts and the output positions of the credited outputs,
encoded as variable length integers. The two line up, so the nth amount belongs to the nth output
index. `federationBtcAddress` says which federation was credited.

The flyover variant is emitted when the outputs are credited to a flyover federation, and carries
the derivation hash that identifies it.

### Coexistence with registerBtcTransaction

`registerBtcTransaction` keeps working and keeps accepting every transaction type, including
peg-outs and migrations.

A release transaction created before the activation went into the RSKIP379 index, not the new one,
and it may still be waiting to be confirmed. So for a window measured in Bitcoin blocks from the
activation height, `registerBtcTransaction` reads the RSKIP379 index first and the new one second.
After the window it reads only the new one.

The window MUST be long enough for every release transaction created before the activation to be
confirmed and registered.

## Rationale

**Why the transaction hash is safe here, when RSKIP379 chose the input sighash instead.** RSKIP379
avoided the transaction hash because signing changed it. That is only true while the signatures the
federation adds are covered by the hash. The federation spends P2SH-P2WSH inputs, whose signatures
live in the witness, and the hash used here excludes the witness, so it is the same before and after
signing.

**Why the outputs are recorded rather than recomputed.** The Bridge could keep the whole transaction
and read its outputs back at registration time. Recording only the outputs that pay back to a
federation stores less, and stores exactly what registering needs.

**Why outputs to other destinations are not recorded.** They are the peg-out payments. The federation never
receives them and can never spend them.

**Why anyone may call it.** As with `registerBtcTransaction`, the caller proves the transaction was
confirmed and the Bridge decides what follows. A caller who submits a hash the Bridge is not
expecting changes nothing.

## Backwards compatibility

This change is a hard fork and therefore all full nodes must be updated.

`registerBtcTransaction` keeps its signature and its behavior for every transaction type. A caller
that keeps using it for peg-outs and migrations continues to work.

The new method does not exist before the activation, so blocks from before the fork replay to the
same state.

## References

[1] [RSKIP379](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP379.md): The peg-out and
migration transaction index this one replaces, and the reasoning about transaction malleability that
made it use an input sighash

[2] [RSKIP305](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md): The change that made
the federation spend P2SH-P2WSH inputs, which is what allows a transaction hash to be used here

[3] [RSKIP378](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP378.md): The release
transaction size limit, which bounds the transaction this method registers

[4] [BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki): Segregated witness,
which moves the signatures out of the data the transaction hash covers

[5] [BIP37](https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki): Partial merkle trees,
the proof format this method takes

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
