---
rskip: 695
title: The snapshot sync protocol
description: An informational description of the snapshot sync wire protocol - its six messages, its parameters, and the order in which a client fetches headers, state and blocks.
status: Draft
purpose: Usa
author: SDL (@SergioDemianLerner), Claude Opus 5
layer: Net
complexity: 2
created: 2026/10/02
---
# The snapshot sync protocol


|RSKIP          | 695 |
| :------------ |:-------------|
|**Title**      |The snapshot sync protocol |
|**Created**    |OCT-2026 |
|**Author**     |SDL, Claude Opus 5 |
|**Purpose**    |Usa |
|**Layer**      |Net |
|**Complexity** |2 |
|**Status**     |Draft |


## Abstract

Snapshot sync lets a node join the network by downloading a recent state trie
directly, instead of executing every block from genesis. It is implemented in
rskj and in rustock, but has never been written down. This RSKIP is
informational: it describes the six messages, the parameters that govern them,
and the order in which a client fetches the three things it needs — the headers
that establish the chain's work, the state, and the most recent blocks.

Nothing here changes consensus. A node that snapshot syncs ends up with the
same state as a node that executed the chain, or it fails.

## The three things a client needs

**The headers**, back far enough to establish how much cumulative work the
checkpoint represents. A checkpoint is a claim by a peer; the work behind it is
what turns the claim into evidence. In RSK this requires the referenced uncle
headers as well, because total difficulty advances by the trunk difficulty plus
every uncle's, and uncle headers travel only in block bodies. Without them a
client is comparing a number it cannot derive.

**The state trie** under the checkpoint block's state root. This is the bulk of
the transfer — tens of gigabytes on mainnet — and it is what the client cannot
compute for itself without executing the whole chain.

**The most recent blocks, with bodies**, so the node can execute forward from
the checkpoint, serve a reorg, and answer the precompiles that read recent
block information. rskj requires `BLOCKS_REQUIRED = 6000`.

## Related proposals

This document describes how a client sequences snapshot sync and why. Three
companions cover the wire itself:

- **RSKIP-696** specifies the six messages below — ids, RLP layouts, and the
  rules a sender and receiver must follow.
- **RSKIP-697** lets a node announce the lowest block it can serve, so a
  requester knows which peers can answer for a given range.
- **RSKIP-698** carries the uncle headers a trunk header references, which is
  what makes the cumulative work behind a checkpoint computable from a header
  walk at all.

## Messages

Six messages, ids 20 to 25, specified in RSKIP-696:

| name | id | carries |
|---|---|---|
| `SNAP_STATE_CHUNK_REQUEST` | 20 | checkpoint block number, byte offset into the trie |
| `SNAP_STATE_CHUNK_RESPONSE` | 21 | a run of trie nodes, and the offset to ask for next |
| `SNAP_STATUS_REQUEST` | 22 | — |
| `SNAP_STATUS_RESPONSE` | 23 | the checkpoint block, the 400 blocks before it, their cumulative difficulties, and the trie's total size |
| `SNAP_BLOCKS_REQUEST` | 24 | a block number to walk down from |
| `SNAP_BLOCKS_RESPONSE` | 25 | 400 blocks and their cumulative difficulties |

### Status

`SNAP_STATUS_RESPONSE` is what makes the rest possible. It names the checkpoint
— a block `checkpointDistance` behind the server's tip, 10,000 by default, so
that the state being offered is settled — and hands over the 400 blocks ending
at it together with their cumulative difficulties. The trie size lets the client
size the transfer before committing to it.

The difficulties are not decoration. A client checks, for each adjacent pair,

```
td(parent) == td(child) - child.cumulativeDifficulty()
```

where `cumulativeDifficulty()` is the block's difficulty plus its uncles'. An
implementation whose stored totals omit the uncle contribution fails this check
and cannot be snapshot synced from, which is how the omission is usually
discovered.

### State chunks

The trie is transferred as a flat, ordered byte stream rather than node by
node. The client asks for a byte offset and receives a run of nodes plus the
next offset, until the stream reaches the size the status response declared.
The nodes are reassembled into a trie and checked against the checkpoint's
state root; a stream that does not reproduce that root is worthless and the
sync fails. A server may bound concurrent requests per peer
(`maxSenderRequests`, 3 by default).

### Blocks

`SNAP_BLOCKS_REQUEST` walks downward in chunks of `BLOCK_CHUNK_SIZE = 400`,
from the checkpoint toward `checkpoint - BLOCKS_REQUIRED`. Each response
carries the blocks and their cumulative difficulties, and the client applies
the same pairwise check as above, plus the ordinary block validation rules.

## Phase order

The three fetches are largely independent — only the checkpoint ties them
together — so an implementation has latitude. The recommended order is
**headers, then state, then blocks**:

1. **Headers first, with their uncles.** This is what converts the peer's
   checkpoint claim into verified cumulative work. It is header-sized traffic,
   cheap beside the state, and it is the step that decides whether the state is
   worth downloading at all. Doing it first means a checkpoint that cannot be
   supported costs a few hundred megabytes of headers rather than tens of
   gigabytes of trie. Ordering it after the state inverts that: the client
   spends its largest transfer before learning whether the claim behind it
   holds.
2. **State second.** It is the bulk of the sync and the part the client cannot
   compute for itself. Once the work behind the checkpoint is established, the
   state root is a commitment the client can verify absolutely, so this
   transfer is the one step that cannot be subverted — but only once step 1 has
   decided which checkpoint to trust.
3. **Blocks last.** They are needed only when the node begins executing
   forward, so they are the last thing that must be in hand, and fetching them
   last keeps the 6,000-block window as close to the tip as possible — a
   window fetched early has already begun to age by the time the state
   finishes.

The ordering is therefore not a performance preference. It is the sequence in
which a client learns enough to justify its next expense.

### What the implementations do today

**rustock** follows this order:
`AwaitingStatus → SamplingClaim → VerifyingHeaders → DownloadingState →
DownloadingBlocks`. The extra `SamplingClaim` phase sharpens step 1 further: it
bounds the peer's claimed difficulty from a few hundred sampled headers before
committing even to the header walk. Measured on mainnet at head #9,288,884,
that walk took **6,971 s and about 20.5 GB** — 10.06 GB of trunk headers back
to genesis and 10.47 GB of the uncle headers they reference, at roughly one
uncle per block. A checkpoint that cannot be supported is abandoned within
seconds rather than after two hours.

The uncle headers are what make it expensive. A client that totals cumulative
work from a header walk must fetch them, because uncle difficulty counts toward
the total and uncle headers exist only in block bodies, which roughly doubles
the header stream. A client that does not total cumulative work does not pay
it.

**rskj** fetches blocks first and the state afterwards: `processSnapStatusResponse`
requests block chunks, and only once `blocksVerified` holds does it call
`generateChunkRequestTasks` and `startRequestingChunks`. Historical headers are
requested alongside the state chunks, and only when `checkHistoricalHeaders` is
set. It defaults to true, but a client that turns it off downloads the whole
state before — in fact instead of — establishing the work behind the checkpoint
it is trusting.

## Parallelism

Snapshot sync is the first RSK sync that invites fetching from several peers at
once, and it is easy to get wrong in ways that fail late and expensively.

### Only one phase is actually parallel

The state is addressed by byte offsets into a fixed object, so any offset can
be requested at any time, from anyone. The other two phases are **chains**: the
next request is defined by the answer to the previous one. A block chunk is
requested by the lowest block already held; a header chunk starts from the
parent hash of the last header verified. Neither can be issued ahead of time,
and no amount of concurrency changes that.

So the shape is: a sequential walk that establishes trust, and a parallel
transfer of the thing being trusted. Attempting to parallelise the walks yields
requests whose answers cannot be checked until the gaps between them are filled,
which is a reordering of the same work rather than a speed-up.

### A block number is not a state

A client fetching state from several peers must ensure they are all answering
the same question. `SNAP_STATE_CHUNK_REQUEST` names a block *number*, and two
peers on different forks each have a block at that height with a different
state root. Both will answer in good faith. The chunks interleave into a trie
that reproduces neither root, and the client learns this only after the whole
state has been transferred.

Binding each request to the expected **state root** is what makes multi-peer
fetching sound; RSKIP-696 specifies the element that carries it and the refusal
a server returns when its root differs. Without such a binding, a client should
fetch the state from a single peer and accept the transfer rate that implies —
which is the choice rskj's `parallel = false` default makes.

### The anchor is not parallel either

Everything a client verifies hangs from one checkpoint, taken from one status
response. Data may come from anywhere; the anchor may not. A client that
accepts a second peer's checkpoint part-way through has verified two chains
against each other and neither against itself.

### Peers are chosen by capability, not availability

These messages go only to peers that announced the `snap` capability. A peer
that did not will not recognise the message id; rskj throws out of
`MessageType.valueOfType` and closes the connection, so spraying requests
across all connected peers costs connections instead of gaining throughput.

The same holds for any message gated on a protocol version — a request for
headers carrying their uncles (RSKIP-698) goes only to a peer that negotiated
the version carrying that message, and a client must keep the plain request as
the fallback for everyone else.

In practice this means a client maintains two sets: peers that can serve a
snapshot, and peers that can serve the extensions it would like. Neither is the
set of connected peers, and a round robin over the latter is a bug.

## Parameters

| name | value | meaning |
|---|---|---|
| `checkpointDistance` / client `limit` | 10000 | how far behind the tip the offered checkpoint sits |
| `BLOCK_CHUNK_SIZE` | 400 | blocks per block-chunk response |
| `BLOCKS_REQUIRED` | 6000 | blocks with bodies a client must end up holding |
| `chunkSize` | 192 | state chunk sizing |
| `maxSenderRequests` | 3 | concurrent client requests a server will queue per peer |
| `checkHistoricalHeaders` | true | whether the client verifies work behind the checkpoint |

## Security notes

A checkpoint is a claim. Everything a client does afterwards is either
verifying that claim or acting on it, and the two must not be confused:

- The state is verified **absolutely**, against the checkpoint's state root. A
  server cannot forge state for a checkpoint the client has accepted.
- The checkpoint itself is verified only **relatively**, by the work behind it.
  This is why the header phase matters and why disabling
  `checkHistoricalHeaders` is a real reduction in security rather than a
  tuning knob — and why, in RSK, the uncle headers are part of the evidence
  rather than an optimisation.
- The 6,000 blocks are validated by the ordinary block rules, so a server
  cannot slip an invalid block into the window.

A client SHOULD bound the claimed work before committing to a long walk, and
SHOULD treat a peer whose difficulties fail the pairwise check as having
offered an unusable snapshot rather than as merely slow.
