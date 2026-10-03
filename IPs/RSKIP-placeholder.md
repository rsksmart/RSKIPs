---
rskip: 697
title: Announce the range of blocks a node serves
description: Let a node state the lowest block it can serve, in the status message and in a new update message, so that peers stop asking pruned nodes for history they have discarded.
status: Draft
purpose: Usa
author: SDL (@SergioDemianLerner), Claude Opus 5
layer: Net
complexity: 1
created: 2026/10/01
---
# Announce the range of blocks a node serves


|RSKIP          | 697 |
| :------------ |:-------------|
|**Title**      |Announce the range of blocks a node serves |
|**Created**    |OCT-2026 |
|**Author**     |SDL, Claude Opus 5 |
|**Purpose**    |Usa |
|**Layer**      |Net |
|**Complexity** |1 |
|**Status**     |Draft |


## Abstract

A node that has discarded old blocks has no way to say so. The status message
carries the best block number, its hash, its parent and the total difficulty,
and nothing about what the node still holds. Peers therefore ask it for history
it threw away, and learn the answer only by waiting out a timeout.

This RSKIP proposes that a node state the lowest block it can serve: as a fifth
element of the status message, and — because that figure changes as a node
prunes — in a new `BLOCK_RANGE_UPDATE` message sent when it moves.

Both are additive. An implementation that does not know about them is unaffected
and keeps working exactly as today.

## Motivation

Two things already in the network make this worth having.

**Snapshot sync.** A node that joins by downloading a state has the headers but
only a window of recent blocks. It cannot serve the rest, and it has no way to
say which rest.

**Block pruning.** A node that bounds its disk by discarding blocks below a
floor is in the same position, and worse: its floor rises continuously, so even
a figure stated once at connection time is wrong minutes later.

In both cases the cost falls on the asking peer, not on the node that cannot
answer. A sync that selects such a peer for a range it does not hold spends a
request and a timeout to discover what one integer would have told it. The peer
is not misbehaving and should not be penalised, so the only feedback is delay.

The same problem was met and solved in Ethereum. `eth/69` replaced total
difficulty in the status message with `earliestBlock`, `latestBlock` and
`latestBlockHash`, and added a `BlockRangeUpdate` message so that a pruning node
can revise the figure without reconnecting. This proposal follows that design,
with one deliberate difference noted below.

## Specification

### 1. Capability

A node that implements this proposal advertises `rsk/63` in its `Hello`,
alongside `rsk/62`.

Capability negotiation is unchanged: each side intersects the peer's
capabilities with its own and takes the highest `rsk` version in common. A node
advertising `rsk/63` to one that does not implement it therefore negotiates
`rsk/62` and behaves exactly as today.

### 2. Status message

The status message gains a fifth element:

```
[ bestBlockNumber, bestBlockHash, bestBlockParentHash, totalDifficulty,
  earliestBlock ]
```

`earliestBlock` is the lowest block number this node can serve. A node holding
the whole chain states `0`.

**Total difficulty is retained.** `eth/69` removed it because the merge made it
meaningless on Ethereum; that is not true here, and removing it would change
what element 3 means for every existing node. The new element is appended, never
substituted.

The element is only present in the four-element form of the message. The
two-element form — `[number, hash]` — is unchanged, and a node sending it says
nothing about what it serves.

### 3. `BLOCK_RANGE_UPDATE` message

A new message type `26`, whose body is:

```
[ earliestBlock, latestBlock, latestBlockHash ]
```

A node sends it when the range it can serve changes — in practice, when a prune
sweep moves its floor. It is sent **only to peers that negotiated `rsk/63`**.

`26` is proposed because it is the first type number not in use: the highest
currently defined is `25` (`SNAP_BLOCKS_RESPONSE`).

### 4. How a receiver must read it

Three rules, each of which matters.

**Silence means everything.** A peer that states no range serves the whole
chain. Every node that has not implemented this proposal looks exactly like
that, and so does a node that has, until its status arrives. Reading an absent
range as "serves nothing" would empty the peer set on a network where most peers
are silent.

**The lower bound is exact; the upper bound is not.** A node never holds blocks
below its stated `earliestBlock`, so that figure can be relied on. `latestBlock`
is a snapshot that trails the sender's real head between announcements, so a
peer merely late with an update should still be asked for recent blocks.

**A range is a hint, not a promise.** A peer may fail to answer within its stated
range — it may have just pruned, or be loaded. Receivers should handle that as
they handle any empty reply today, and must not treat it as misbehaviour.

### 5. Compatibility

Both parts are additive and have been verified against rskj 9.1.0.

The status message decoder in `MessageType.STATUS_MESSAGE.createMessage` reads
indices 0 through 3 and consults the list length only to distinguish the
two-element form. A five-element message takes the same branch a four-element
one does, and the fifth element is ignored. There is no length validation at any
layer above it: `Eth62MessageFactory` unwraps and `Message.create` dispatches,
neither checking arity.

This was confirmed by decoding two-, four-, five- and six-element status
messages through rskj's own classes, including messages produced by an
independent encoder. All were accepted, with the known fields parsed identically:

```
four-element   ACCEPTED  best=#9000000  parent=set  td=123456789
extended       ACCEPTED  best=#9000000  parent=set  td=123456789
```

The new message type is a different matter, and is the reason for the capability
gate. `MessageType.valueOfType` throws `IllegalArgumentException` for a type it
does not know, and `P2pHandler.exceptionCaught` answers that with `ctx.close()`.
Sending `BLOCK_RANGE_UPDATE` to a node that has not negotiated `rsk/63` would
therefore drop the connection. **Implementations must gate the message on the
negotiated version, not on optimism.**

## Rationale

### Why not only the status message

A pruning node's floor rises with its head. A node that prunes to a depth of
8,000 blocks moves its floor roughly as fast as the chain advances, so a figure
fixed at connection time is stale within minutes and wrong by thousands of
blocks within a day. Peers rarely reconnect. Without an update message the
status element would describe a node as it was when the connection opened, which
is worse than useless on the nodes that most need it.

### Why not infer it from failed requests

A receiver could record which peers failed to answer which ranges. This is what
implementations do today, and it works — at the cost of one request and one
timeout per discovery, repeated per range, re-learned after every reconnection,
and indistinguishable from a peer that is merely slow.

### Why append rather than replace

Replacing total difficulty, as `eth/69` did, would change the meaning of element
3. Every node that had not been upgraded would read a block number as a
difficulty, and act on it. Appending costs a handful of bytes per connection and
is invisible to anything that does not look for it.

## Backwards compatibility

A node implementing this proposal and one that does not interoperate exactly as
two unimplementing nodes do today:

- The extra `rsk/63` capability is dropped during negotiation.
- The fifth status element is ignored by the decoder.
- `BLOCK_RANGE_UPDATE` is never sent, because the version was not negotiated.

No activation height is required. No consensus rule changes. A node may
implement the reading half without the writing half, or the reverse.

## Reference implementation

Implemented in rustock: the capability and negotiation, the status element, the
`BLOCK_RANGE_UPDATE` message, and the peer-selection change that consults the
range when assigning work.

Two nodes were run against each other, and against a node speaking the current
protocol. The observed result, from the node that implements the proposal:

```
peer [153, 70, 158, 6] speaks rsk/63 snap/1, serves from #9278157
peer [89, 41, 78, 237] speaks rsk/62, served range unstated
```

The first is a pruned peer stating a real floor; the second is a node on the
current protocol, correctly read as having said nothing.

## A related gap

rskj defines `Capability.SNAP = "snap"` with `SNAP_VERSION = 1`, advertises it
whenever it speaks RSK, and looks for it in a peer's hello. It is already the
mechanism by which a node says whether it has anything to do with snapshots, and
it needs no proposal — only for other implementations to advertise it too, which
rustock now does. It is mentioned here because it answers the adjacent question,
"will this peer serve me a snapshot", by the same means and should not be
reinvented.

## Open questions

1. **Should `latestBlock` be in the status message as well?** The status message
   already carries `bestBlockNumber`, which is the same figure, so this proposal
   adds only `earliestBlock` there and carries all three in the update message.
   The asymmetry is deliberate but worth confirming.

2. **Should a node announce on a schedule as well as on change?** A node whose
   floor is static says nothing after the handshake, which is correct but gives a
   peer no way to detect a stale record. A periodic re-announcement would cost
   little.

3. **Is one update per sweep the right granularity?** A node pruning in large
   batches announces rarely and in large steps; one pruning continuously would
   announce often. A minimum interval may be worth specifying.
