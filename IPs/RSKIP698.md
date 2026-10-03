---
rskip: 698
title: Send referenced uncle headers alongside trunk headers
description: A new header message that carries each trunk header together with the uncle headers it references, so that a node syncing headers alone can compute and validate cumulative work.
status: Draft
purpose: Usa
author: SDL (@SergioDemianLerner), Claude Opus 5
layer: Net
complexity: 2
created: 2026/10/01
---
# Send referenced uncle headers alongside trunk headers


|RSKIP          | 698 |
| :------------ |:-------------|
|**Title**      |Send referenced uncle headers alongside trunk headers |
|**Created**    |OCT-2026 |
|**Author**     |SDL, Claude Opus 5 |
|**Purpose**    |Usa |
|**Layer**      |Net |
|**Complexity** |2 |
|**Status**     |Draft |


## Abstract

In RSK, total difficulty advances by the trunk block's difficulty **plus the
difficulty of every uncle it references**. The uncle headers travel only in the
block body. A node that downloads headers alone therefore cannot compute the
chain's cumulative work, and cannot validate the work claimed by a peer.

This RSKIP proposes a header message that carries, for each trunk header, the
uncle headers that header references. The uncle list is already committed by
`unclesHash`, so the attachment is self-verifying: a peer can neither add,
drop, nor alter an uncle without breaking the commitment in a header whose
proof of work the receiver checks.

It is additive. A node that does not implement it keeps using the existing
header message unchanged.

## Motivation

### Cumulative work is not derivable from trunk headers

Ethereum advances total difficulty by the trunk header's difficulty alone;
uncles affect rewards, not work. A header chain is therefore sufficient to
compute and compare cumulative work, which is what makes header-only light
sync possible.

RSK diverged. `Block.getCumulativeDifficulty()` is

```java
BlockDifficulty calcDifficulty = this.header.getDifficulty();
for (BlockHeader uncle : uncleList) {
    calcDifficulty = calcDifficulty.add(uncle.getDifficulty());
}
```

so uncle difficulty is part of the chain's work. The trunk header records
`uncleCount` and commits to the uncle list through `unclesHash`, but carries
neither the uncles nor their difficulties. The consequence is strict: **in RSK,
a node holding every trunk header from genesis still cannot say how much work
the chain represents.**

### The difficulty cannot be reconstructed from the canonical chain

An uncle is a sibling of a canonical block — it shares that block's parent.
It is tempting to assume siblings share a difficulty, which would let a node
recover an uncle's difficulty from the canonical header at the same height.
They do not. `DifficultyCalculator.getBlockDifficulty` keys off the parent's
difficulty **and the candidate block's own timestamp and own uncle count**:

```java
long parentBlockTS = parent.getTimestamp();
int  uncleCount    = curBlockHeader.getUncleCount();
long curBlockTS    = curBlockHeader.getTimestamp();
long delta         = curBlockTS - parentBlockTS;
int  calcDur       = (1 + uncleCount) * duration;
```

Measured on mainnet across 22,784 uncles sampled in every consensus era: every
uncle is a true sibling of the canonical block at its height (0 exceptions),
and **2,502 of them — 11% — carry a different difficulty from that sibling**.
A worked case, uncle 0 of block #9,280,005:

| | timestamp | uncles | difficulty |
|---|---|---|---|
| canonical #9,280,003 | +38s from parent | 2 | 7206481898605743909862 |
| its sibling, the uncle | +43s from parent | 0 | 7170539345495490823030 |

So the information is genuinely absent from the header stream. It is not a
matter of implementations failing to look.

### What this costs today

Every participant that would otherwise work from headers has to compensate:

- A light client cannot validate cumulative work at all. There is no accepted
  design for a light client that does not — proof of work *is* the security
  argument, and a client that cannot weigh it is trusting its peer.
- A full node doing header-first sync computes total difficulty at the header
  stage and is obliged either to understate it, or to revisit it once bodies
  arrive. rskj avoids the question by computing total difficulty only in
  `BlockChainImpl.tryToConnect`, after the body is in hand — its header and
  body download states never mention difficulty. That works, but it means
  header sync carries no work semantics at all and cannot be used to choose
  between chains.
- A node that has pruned its bodies can never recompute the figure, because
  the uncle headers were in the bodies and nowhere else. The freezer keeps
  canonical headers by height, and an uncle is by definition not canonical.

## Specification

### Messages

Two messages are added to the RSK protocol at version `63`:

| name | id | payload |
|---|---|---|
| `BLOCK_HEADERS_WITH_UNCLES_REQUEST` | 27 | `[id, hashOrNumber, maxHeaders, skip, reverse]` |
| `BLOCK_HEADERS_WITH_UNCLES_RESPONSE` | 28 | `[id, [entry...]]` |

The request mirrors the existing header request so that a node may substitute
one for the other without changing its sync logic.

Both are additive. A reference implementation exists in rustock, an
independent RSK node.

Each `entry` is

```
entry = [ trunkHeader, [uncleHeader, ...] ]
```

where `trunkHeader` is encoded exactly as in the existing header message, and
the uncle list is the block's `uncleList` in its original order — the same
sequence `unclesHash` commits to.

A responder MUST send the complete uncle list for every entry, or omit the
entry. A responder that no longer holds the body for a block, and therefore
cannot produce its uncles, MUST NOT substitute an empty list; it stops the
response at that entry. Combined with RSKIP-697, a requester knows in
advance which peers can answer.

### Validation

On receiving an entry the node MUST check, in this order:

1. `keccak256(RLP(uncleList)) == trunkHeader.unclesHash`.
2. `len(uncleList) == trunkHeader.uncleCount`.
3. Proof of work on `trunkHeader`.
4. Proof of work on each uncle header.
5. Each uncle's parent is an ancestor of the trunk header within
   `uncleGenerationLimit`, and the uncle's difficulty is the difficulty its
   parent implies, by the ordinary difficulty rule.

Only then may it accumulate

```
cumulative(n) = trunkHeader(n).difficulty + sum(uncleHeader.difficulty)
td(n)         = td(n-1) + cumulative(n)
```

Check 1 is what makes the message trustless: the uncle list is fixed by a
commitment inside a header whose proof of work the node has verified. Check 4
is what stops a peer claiming cheap work — without it, a miner could commit to
uncles bearing arbitrary difficulty and inflate the chain's apparent weight for
the cost of a single trunk header. rskj already applies both, through
`BlockUnclesHashValidationRule` and `BlockUnclesValidationRule`, the latter
carrying `getProofOfWorkRule()`.

Check 5 is possible for a header-only node precisely because an uncle's parent
lies on the trunk it is already syncing, within `uncleGenerationLimit` blocks.

Check 1 MUST be computed over the uncle list **as it arrived on the wire**, not
over a re-encoding of the decoded headers. An implementation that re-encodes is
comparing its own spelling of the bytes against a commitment the miner made
over theirs, and the two need not agree. This is the same hazard that obliges a
node to keep original transaction bytes when rebuilding a transaction trie.

A responder MUST NOT send a pairing it cannot itself justify: if its stored
uncles do not reproduce the header's `ommers_hash`, the run ends there.

### Capability negotiation

A node offering `rsk/63` announces support. A node at `rsk/62` is unaffected
and continues to be served the existing header message.

## Cost

Uncle headers must be sent whole, because their proof of work is validated and
that requires the merged-mining fields. Measured on a mainnet node:

| quantity | value |
|---|---|
| average RSK header | 1,085 bytes |
| uncles per trunk block (11 eras × 3,000 blocks) | 1.04 |
| full-chain header sync today | 10.1 GB |
| full-chain header sync with uncles | 20.5 GB |

So the header stream roughly doubles. That is the honest price of making
cumulative work verifiable from headers, and it is only paid by a node that
asks for the new message. A node syncing from a trusted checkpoint does not
need the history at all, and a node that wants the chain's work without
trusting anyone could not previously obtain it at any price.

## Backwards compatibility

Additive. The existing header message is unchanged and remains the default.
Nothing in consensus changes: the quantities carried are ones every full node
already computes and validates.

## Rationale, and what was rejected

**Deriving uncle difficulty from canonical siblings.** Measured wrong — 11% of
uncles differ from their canonical sibling. See above.

**Computing total difficulty only after bodies arrive, as rskj does.** It is
correct, and it is what rskj does today, but it concedes the point: header sync
then carries no work semantics, so it can never serve a light client, and a
node cannot rank chains until it has downloaded and executed bodies.

**Carrying the uncle difficulties, rather than the uncle headers.** Smaller,
and unsound: a difficulty that is not attached to a proof of work is a number
the peer asserts. The commitment in `unclesHash` is over headers, so headers
are what must be sent.
