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

A peg-out or a migration transaction is registered in the Bridge so that the funds it pays back to a
federation, the change of a peg-out and the funds moved by a migration, can be spent again. Today
that is done by submitting the whole serialized transaction to `registerBtcTransaction`, which is
redundant: the Bridge built that transaction, so it already knows its outputs.

This RSKIP has the Bridge record, when it creates a release transaction, the outputs of that
transaction that pay back to a federation, under the transaction hash. It then adds a method that
registers such a transaction from that hash and a proof that it has enough Bitcoin confirmations.

It also adds two events reporting the outputs credited to a federation, and starts recording the
Bitcoin block height of each of those outputs, which until now was always stored as zero.

## Motivation

Registering a release transaction today means sending the Bridge back the bytes it produced itself.
It picked the inputs and built the outputs, so none of them tell it anything new. The one thing it
cannot know is whether the transaction reached Bitcoin and is buried deep enough, which is what the
proof is for.

Sending the whole raw transaction anyway makes the RSK transaction that does the registering grow
with the Bitcoin transaction it carries, so what a registration takes is only known once the release
transaction exists. A hash is always the same 32 bytes, so every registration is the same size,
known ahead of time, and identical for a peg-out with one input and for a migration with many.

## Specification

### The new method

```solidity
function registerPegoutTransaction(bytes32 btcTxHash, int height, bytes calldata pmt) external;
```

The method exists only once `RSKIP643` is active, and anyone may call it. `height` and `pmt` mean
what they mean in `registerBtcTransaction` [1] and have the same types, so a caller that can build a
proof for one can build it for the other.

`btcTxHash` is the transaction hash without the witness. The call does not revert and changes no
state when any of the following holds:

1. The hash has already been registered.
2. The height does not have enough confirmations, or the partial merkle tree does not prove the hash
   at that height.
3. The hash is not in the index described below.

The third covers anything that is not a release transaction the Bridge created, a peg-in included or any random bitcoin transaction.


Otherwise the Bridge credits the outputs recorded for that hash to the federation each one pays to,
removes the entry from the index, and marks the hash as registered.

The call has a fixed cost, as `registerBtcTransaction` does.

### The index

When the Bridge creates a release transaction it records, under that transaction's hash, the outputs
of that transaction that pay to the active or the retiring federation. For each one it stores what
registering it needs: the amount, the position of the output in the transaction, and the script it
pays to, which is what says which federation to credit. Outputs paying anywhere else are not
recorded, since the federation never receives them.

The entry is keyed by the transaction hash, under the storage key `federationsPendingBtcUTXOs`, one
entry per transaction, so an entry is read and removed without touching any other.

An entry is never empty. A release transaction that pays nothing back to a federation is not
recorded, so its hash is not in the index and trying to register it does nothing, like any other hash the
index does not hold.

This index is how the Bridge now recognizes its own release transactions, in place of the peg-out
transaction index of RSKIP379 [2]. That one is still read during the transition described below.

### The height of credited outputs

Every output credited to a federation is stored with the height of the Bitcoin block that confirmed
it, so the Bridge can tell how old each of its outputs is. Until now that height was always recorded
as zero.

The height is the `height` argument of the registration call, for both methods, which the proof ties
to the block containing the transaction.

### Events

Two events are added. Both are emitted whenever outputs are credited to a federation, whichever
method did it, and only once `RSKIP643` is active.

```solidity
event utxos_registered(bytes32 indexed btcTxHash, bytes values, bytes outputIndexes, string federationBtcAddress);
event flyover_utxos_registered(bytes32 indexed btcTxHash, bytes values, bytes outputIndexes, string federationBtcAddress, bytes32 flyoverDerivationHash);
```

`values` holds the amounts of the credited outputs in satoshis and `outputIndexes` their positions
in the transaction. Each field is the concatenation of those numbers encoded as Bitcoin compact size
integers, each in its shortest form. Both fields hold the same count of numbers in the same order,
so the nth amount belongs to the nth position. `federationBtcAddress` says which federation was
credited.

The flyover variant is emitted when the outputs are credited to a flyover federation, which only a flyover
peg-in does, and carries the derivation hash that identifies it.

### Coexistence with registerBtcTransaction

`registerBtcTransaction` keeps working and keeps accepting every transaction type, including
peg-outs and migrations.

From the activation on, every release transaction the Bridge creates goes into the new index, and
the RSKIP379 index is no longer written. A release transaction created before the activation went
into the RSKIP379 index instead, and it may still be waiting to be confirmed.

So up to a cutoff, `registerBtcTransaction` looks for the transaction in the new index first and in
the RSKIP379 index second, and from the cutoff on it looks only in the new index.
`registerPegoutTransaction` only ever looks in the new index.

A caller therefore does not have to tell the two apart while the window is open.
`registerBtcTransaction` covers release transactions created on either side of the activation, so a
caller that does not know which side a given transaction falls on can keep using it.

The cutoff is a Bitcoin block height, fixed per network, and applies to transactions confirmed at
that height or above.

The cutoff MUST be high enough for every release transaction created before the activation to be
confirmed and registered before it is reached. After the cutoff neither method recognizes one of
those transactions, and the funds it pays back to a federation stay out of the Bridge's
accounting.

## Rationale

**Why the transaction hash is safe here, when RSKIP379 chose the input sighash instead.** RSKIP379
avoided the transaction hash because signing changed it. That is only true while the signatures the
federation adds are covered by the hash. Federations spend P2SH-P2WSH inputs, whose signatures live
in the witness, and the hash used here excludes the witness, so it is the same before and after
signing.

**Why the outputs are recorded rather than recomputed.** The Bridge could keep the whole transaction
and read its outputs back at registration time. Recording only the outputs that pay back to a
federation stores less, and stores exactly what registering needs.

**Why outputs to other destinations are not recorded.** They are the peg-out payments. The
federation never receives them and can never spend them.

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

[1] [RSKIP40](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP40.md): The two-way peg Bridge,
where `registerBtcTransaction` and its parameters are defined

[2] [RSKIP379](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP379.md): The peg-out and
migration transaction index this one replaces, and the reasoning about transaction malleability that
made it use an input sighash

[3] [RSKIP305](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP305.md): The change that made
federations spend P2SH-P2WSH inputs, which is what allows a transaction hash to be used here

[4] [RSKIP378](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP378.md): The release
transaction size limit, which bounds the transaction this method registers

[5] [BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki): Segregated witness,
which moves the signatures out of the data the transaction hash covers

[6] [BIP37](https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki): Partial merkle trees,
the proof format this method takes

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
