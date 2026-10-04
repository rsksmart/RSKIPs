---
rskip: 696
title: Snapshot sync wire messages
description: Specification of the six snapshot sync messages already in use - their ids, their RLP layouts, and the rules a sender and a receiver must follow.
status: Draft
purpose: Usa
author: SDL (@SergioDemianLerner), Claude Opus 5
layer: Net
complexity: 2
created: 2026/10/03
---
# Snapshot sync wire messages


|RSKIP          | 696 |
| :------------ |:-------------|
|**Title**      |Snapshot sync wire messages |
|**Created**    |OCT-2026 |
|**Author**     |SDL, Claude Opus 5 |
|**Purpose**    |Usa |
|**Layer**      |Net |
|**Complexity** |2 |
|**Status**     |Draft |


## Abstract

Six messages carry snapshot sync. They are implemented and in use, and have
never been specified. This RSKIP writes down what is on the wire today: the
message ids, the RLP layout of each, and the rules a sender and a receiver must
follow.

It describes existing behaviour and proposes no change to it. RSKIP-695
describes how a client sequences these messages and why; this document is the
format alone.

## Framing

Every message here carries a request id, and is framed identically:

```
message = RLP([ id, params ])
```

where `id` is an unsigned integer and `params` is the RLP-encoded list given
below for that message type. The `params` element is an RLP *string* holding an
encoded list, not a nested list: a receiver decodes `params` and then decodes
its contents again. An implementation that writes a nested list instead
produces a message rskj cannot parse, and the failure is silent — the request
simply never arrives.

A response MUST carry the id of the request it answers. A requester matches
answers by id and MUST ignore an id it is not expecting.

## Messages

| name | id |
|---|---|
| `SNAP_STATE_CHUNK_REQUEST` | 20 |
| `SNAP_STATE_CHUNK_RESPONSE` | 21 |
| `SNAP_STATUS_REQUEST` | 22 |
| `SNAP_STATUS_RESPONSE` | 23 |
| `SNAP_BLOCKS_REQUEST` | 24 |
| `SNAP_BLOCKS_RESPONSE` | 25 |

### `SNAP_STATUS_REQUEST` (22)

```
params = RLP([])
```

Carries nothing: the request is the question. A server that does not serve
snapshots does not answer.

### `SNAP_STATUS_RESPONSE` (23)

```
params = RLP([ [block, ...], [difficulty, ...], trieSize ])
```

- `block` — a whole block, header and body, encoded as blocks are elsewhere
  on the wire.
- `difficulty` — the cumulative difficulty **at** the block of the same index.
  The two lists MUST be the same length and in the same order.
- `trieSize` — the total size in bytes of the state the server is offering,
  so a client can size the transfer before committing to it.

The newest block in the list is the checkpoint. The server also sends the
blocks immediately below it — 400 in the current implementation — so that the
client can check the chain's shape before trusting anything.

A client MUST verify, for each adjacent pair, that

```
difficulty[i-1] == difficulty[i] - cumulativeDifficulty(block[i])
```

where `cumulativeDifficulty` is the block's own difficulty **plus the
difficulty of every uncle it references**. Checking against the header
difficulty alone rejects every honest peer, because an RSK chain absorbs
roughly one uncle per block.

### `SNAP_BLOCKS_REQUEST` (24)

```
params = RLP([ blockNumber ])
```

Asks for the run of blocks ending just below `blockNumber`.

### `SNAP_BLOCKS_RESPONSE` (25)

```
params = RLP([ [block, ...], [difficulty, ...] ])
```

The same pairing as the status response, and the same verification applies. The
current implementation answers with 400 blocks.

A client walks downward by issuing a new request for the lowest block number it
has received, until it holds the blocks it requires — 6,000 in the current
implementation, enough to serve a reorg and to answer the precompiles that read
recent block information.

**Blocks are identified by number but MUST be validated by hash.** The
`blockNumber` in a request tells the server where to read; it does not bind
what comes back. A server whose chain reorganises between two requests will
answer the second from a different chain, at the same heights, in good faith.

A client MUST therefore check that each returned block links by parent hash to
the chain it already holds, anchored at the checkpoint hash from the status
response, and MUST fail the sync rather than retain a run that does not link. A
client that trusts the height alone will splice two chains together and arrive
at a state root that matches neither.

Two implementations do this differently and both are sufficient: one carries
the lowest block held into the next chunk and requires the incoming block to be
its parent; the other checks each block against the header chain it verified
for itself, which is stronger because it binds the body to an independently
established chain rather than only to the previous answer.

Note that the checkpoint distance does **not** protect against this. A
checkpoint 10,000 blocks behind the tip is deep enough that the *state* being
offered is settled and will still be there when the transfer finishes, which is
what that distance is for. The blocks below it are still served from whatever
the server's canonical chain says at request time.

### `SNAP_STATE_CHUNK_REQUEST` (20)

```
params = RLP([ blockNumber, from, chunkSize ])
```

- `blockNumber` — the checkpoint whose state is being requested.
- `from` — the byte offset into the state stream.
- `chunkSize` — retained for compatibility and ignored by the current server,
  which sizes its own responses. A sender SHOULD still populate it.

**A block number does not identify a state.** Two peers on different forks
both have a block at height `blockNumber`, with different state roots, and both
will answer this request in good faith with their own state. Nothing in the
three fields above distinguishes them.

A client that fetches chunks from a single peer discovers a mismatch only at
the end, when the reassembled trie fails to reproduce the checkpoint's state
root, having downloaded the whole state to find out. A client that fetches
chunks from *several* peers in parallel has no way to ask them the same
question at all, and will interleave two states into a trie that reproduces
neither.

rustock therefore appends a fourth element, the state root the client expects:

```
params = RLP([ blockNumber, from, chunkSize, stateRoot ])
                                             ^^^^^^^^^
```

A server that has a different state root at that height answers with refusal
code 2 (`StateRootMismatch`) instead of serving its own state, so the client
learns immediately and from the peer itself. rskj ignores the element, so a
request carrying it is still answered normally by a server that does not
implement it — in which case the root check at the end remains the only
defence.

### `SNAP_STATE_CHUNK_RESPONSE` (21)

```
params = RLP([ chunkOfTrieKeyValue, blockNumber, from, to, complete ])
```

- `chunkOfTrieKeyValue` — the run of trie nodes, as an RLP string.
- `blockNumber` — echoes the request.
- `from`, `to` — the byte range this chunk covers. `to` is where the client
  asks next.
- `complete` — `1` when this chunk ends the state stream, `0` otherwise,
  encoded as an integer.

The state is transferred as a flat, ordered byte stream rather than node by
node, so a client requests offsets and not hashes. It reassembles the nodes
into a trie and checks the result against the checkpoint's state root. A stream
that does not reproduce that root is worthless, and the sync fails rather than
retaining part of it.

## Extension by trailing elements

rskj's decoders read a fixed number of elements from each message and do not
consult the list length again. An element appended after the last one they read
is therefore accepted and ignored, which makes these messages extensible
without a flag day: a node that understands an addition benefits, and one that
does not is unaffected.

This property is already relied on. RSKIP-697 appends a fifth element to the
status message by the same means.

One extension exists today on `SNAP_STATE_CHUNK_RESPONSE`, added by rustock:

```
params = RLP([ chunkOfTrieKeyValue, blockNumber, from, to, complete, refusal ])
                                                                    ^^^^^^^
```

`refusal` is an integer saying why the payload is empty:

| value | meaning |
|---|---|
| 0 | not a refusal — the chunk is in the payload |
| 1 | no block at that number on this chain |
| 2 | this chain has a different state root at that height |
| 3 | the state was known but is no longer stored |
| 4 | the offset is past the end of the trie |
| 5 | the offset is not a multiple of this server's chunk granularity |

Without it an empty payload is ambiguous: a client cannot tell "I have nothing
for that offset, ask elsewhere" from "I have nothing for you at all" from "you
asked for something malformed", and must guess which by retrying. None of these
is misbehaviour, so none should be charged against the peer; the value only
tells the client whether another request to the same peer is worth a round
trip.

A client MUST NOT depend on this element being present, and MUST treat its
absence as "no reason given". A server that does not implement it simply sends
the five elements rskj sends.

The `stateRoot` element on `SNAP_STATE_CHUNK_REQUEST` above is the second such
extension, and is what makes requesting chunks from more than one peer sound.

These are documented here because they are on the wire, not to propose that
other implementations adopt them.

## Choosing peers

Snapshot sync is not a conversation with one peer, and it is not a round robin
over all of them either.

**The anchor comes from one peer; the data may come from any.** A client takes
one `SNAP_STATUS_RESPONSE`, fixes the checkpoint it names, and validates
everything afterwards against that: the state against its state root, the
blocks by parent-hash linkage to it, the headers by proof of work and linkage.
Provenance of the data does not matter, because none of it is believed on the
sender's word. A client MUST NOT re-anchor to a different peer's checkpoint
part-way through; it either completes against the checkpoint it chose or starts
again.

**Requests are constrained by capability, not availability.** A client MUST
send these six messages only to a peer that announced the `snap` capability
during the handshake. A peer that did not will not recognise the message id,
and rskj throws out of `MessageType.valueOfType` and closes the connection, so
an indiscriminate round robin loses peers rather than spreading load. The same
applies to any protocol-version-gated message: it goes only to a peer that
negotiated the version carrying it.

**What can and cannot be parallelised:**

| | |
|---|---|
| state chunks | parallel across peers, **provided** each request binds the state root; without that binding, parallel fetching is unsound rather than merely risky |
| block chunks | sequential by nature — each request names the lowest block held, so the next cannot be issued until the previous answer links |
| historical headers | sequential by nature — each chunk starts from the parent hash of the last verified header |

Only the state is embarrassingly parallel, because only the state is addressed
by offsets into a fixed object rather than by a chain that must be walked. Both
other phases are chains: their requests are defined by the answer to the
previous one.

A client SHOULD retry a failed or refused chunk against a different peer rather
than the same one, and SHOULD treat a peer whose answers fail validation as
unusable for this sync rather than charging it with misbehaviour — being on
another fork is not a fault.

## Rules

A **server** MUST NOT answer with a range it cannot produce in full. A server
that has pruned the bodies behind a block cannot serve that block, and SHOULD
end the run rather than send a short or empty answer, which a client cannot
distinguish from "there is nothing there".

A server MAY bound concurrent requests per peer. The current implementation
allows three.

A **client** MUST treat every figure in these messages as a claim until it is
checked:

- the state is verified **absolutely**, against the checkpoint's state root;
- the blocks are verified by the ordinary block rules;
- the checkpoint itself is verified only **relatively**, by the work behind it,
  which is what the header phase in RSKIP-695 exists to establish.

The `trieSize` in the status response deserves naming separately: it is a
**hint**, not a fact. A client MUST size nothing from it and MUST decide
completion from the reassembled state reproducing the checkpoint's state root,
not from having received the number of bytes a peer announced.

The cumulative difficulties are claims of the same kind. The pairwise check
above proves they are *self-consistent*; it does not prove the chain carrying
them exists. RSKIP-695 describes bounding the claim against a shipped
checkpoint before committing to the header walk, which is how a client can
refuse an impossible claim in seconds rather than after hours of walking.

A client SHOULD treat a peer whose difficulty pairing fails as having offered
an unusable snapshot rather than as merely slow, and go to another peer.

## Encoding note

Difficulties in these messages are written as rskj writes them everywhere:
`BigInteger.toByteArray()`, which is two's complement, so a value whose top
bit would otherwise be set carries a leading `0x00`. They are read back with
`new BigInteger(bytes)`, which rejects a negative. An implementation that
encodes them as canonical RLP integers — stripping that leading zero — is
correct by the RLP specification and unreadable here, but only once a value's
top bit is set, so the failure appears years into otherwise working
interoperation. Mainnet's cumulative difficulty is in that range now.

## Backwards compatibility

None required: this describes messages already in use.
