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
- **RSKIP-699** commits that cumulative work to the header extension. It is
  under reconsideration and **should not be read as removing the need for the
  uncle headers**: a field stating a total is data, not work, and a miner can
  write any value into a header it mined. Only RSKIP-698 makes the work
  provable.

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

## Bounding the claim before the walk

The header phase is 92% of a snapshot sync. A peer whose claim cannot possibly
be true should cost seconds to reject, not hours — and the cost of finding out
is what decides whether a client can afford to be picky about who it syncs
from.

This section describes the approach one implementation uses. It is not required
by the protocol, and a client that simply walks is correct; it is written down
because the alternative to having something like it is doing the full walk for
every peer that offers a checkpoint.

### The shape of the problem

A peer advertises a cumulative difficulty. Nothing in the status message makes
that figure true, and sync decisions are made from it. The question is whether
the claim is *possible* — not whether the peer is honest, and not which chain
it is on.

### A shipped checkpoint

The client ships a `(height, hash, cumulative difficulty, difficulty)` record
for a block well below the tip. Below that height the chain's work is known
exactly, so nothing may claim more for it. That splits the problem in two:

| | |
|---|---|
| at or below the checkpoint | answered by subtraction, no requests |
| above it | needs evidence from the peer |

The free half is not a defence on its own. A peer claiming a height *above* the
checkpoint sidesteps it for nothing — and that is what a peer wanting to be
chosen as a sync source would claim anyway, since it wants to look like the
longest chain. It is worth having because it costs nothing, not because it
stops an adversary.

### Sampling the window above it

```
  draw K heights above the checkpoint uniformly at random
  add evenly spaced heights until no gap exceeds N blocks
  ask the peer for a header at each, and verify its proof of work
  allowance = uncle bound implied by the sampled uncle counts   (see below)
  for each gap between consecutive samples:
      bound the work those unseen blocks can carry   (see below)
  ceiling = checkpoint work + allowance * sum of the gap bounds
  refuse any claim above the ceiling
```

The two halves of the selection do different jobs and must not be confused.
The **random draw** is what the uncle bound rests on: the peer is committed to
its chain before it learns where it will be checked, so the heights it is asked
about are a uniform sample of a population it has already fixed. A grid of
`checkpoint + k*N` destroys that -- a peer mines real uncles at those heights
and fabricates between them. The **gap filling** proves nothing about uncles;
it exists so that no stretch is wide enough for the difficulty bound below to
compound out of usefulness, and it is a deterministic function of the draw.
Counting the filled positions toward the sample count would overstate what the
method proves.

Fill gaps with evenly spaced points rather than by repeated bisection.
Bisection produces only power-of-two subdivisions, so a gap just over `N * 2^m`
costs nearly twice the requests it needs to.

The samples must come **from the peer being judged**. If its chain is
fabricated, nobody else holds those blocks, so a peer that cannot produce them
has failed by another route. A client should give up on one that will not
answer: refusing to substantiate a claim is not better than being unable to.

### Why a gap can be bounded without seeing it

Difficulty cannot move freely between blocks — the retarget rule limits each
step to `1/divisor`. So given two sampled difficulties, every block between
them faces two ceilings: it cannot rise more than `1/divisor` above its
predecessor, and it cannot be so high that even falling as fast as the rule
allows it would overshoot the next sample. **The lower of the two is the most
that block could have been**, and summing those maxima bounds the gap.

Two details matter for correctness:

- Take the lower of the two ceilings rather than modelling a rise-then-fall
  shape. Consensus permits difficulty to stay *unchanged*, and a shape that is
  always rising or falling cannot express that; for two samples of equal
  difficulty it has no valid path at all, because a maximal rise and a maximal
  fall multiply to `1 - 1/divisor²` and never cancel.
- A difficulty floor clamps the falling side exactly as consensus does. The
  clamp only ever raises a value, which keeps the result an upper bound.

Past the newest sample there is no later difficulty to aim for, so the bound
there is unconstrained growth — which is why the newest sample should sit close
to the claimed head.

### The uncle trap

A sampled header carries its own difficulty. The quantity being bounded is
**cumulative** difficulty, which in RSK is the header difficulty *plus every
uncle the block references* — so a bound computed from sampled header
difficulties is bounding the wrong quantity, and it is bounding a smaller one.

Measured on mainnet over 276,000 blocks above the checkpoint: uncles add
**90.9%** to the work. A ceiling of 1.68× the header-difficulty work in that
window is only **0.88×** the real work — below the honest chain, which means
every truthful peer is judged impossible and the node bounds itself out of the
network.

Two consequences are worth stating plainly, because both are counter-intuitive:

- **Tightening the sampling interval makes it worse.** A shorter interval gives
  a tighter bound on header difficulty, which is the obvious optimisation, and
  it drives the ceiling further below the honest chain. The improvement is the
  failure.
- **A rise in the uncle rate does the same thing**, with no code change at all.
  The rate is not a constant: it was 51.7% over a 30,000-block window in 2026
  and 90.9% over the wider window measured since.

An implementation must therefore bound the uncle contribution rather than
ignore it.

#### Bounding it

Consensus caps uncles per block (`uncleListLimit`, 10 in rskj), so a sound
allowance always exists: assume ten everywhere, and multiply every gap bound by
11. That is correct and nearly useless — roughly six times the honest chain's
work on current mainnet figures.

Sampled headers also carry `uncleCount`, which is inside what the block's proof
of work commits to and so cannot be overstated for a block that was looked at.
Because the heights were drawn at random from a population the peer had already
fixed, the sampled counts are a uniform sample of it, and a concentration
inequality turns them into a bound on the mean. Empirical Bernstein is the
right one here — mainnet's uncle counts are tightly clustered against a range
of 10, and it pays for the observed variance in the square-root term rather
than for the range:

```
  mean <= mean_hat + sqrt(2 * V_hat * L / K) + 3 * R * L / K,   L = ln(3/delta)
  allowance = 1 + min(that, R)
```

**What is and is not being claimed.** Not that any particular stretch of the
chain is free of uncles: a peer can hold a run of ten-uncle blocks between two
samples and no amount of sampling will see it. What the ceiling needs is
weaker. It is a bound on the *sum* over every gap, and the allowance multiplies
every gap alike, so the quantity that has to be bounded is the population mean.
A stretch running hot is paid for by the stretches that do not.

#### What it is worth

Against measured mainnet uncle counts, at `delta = 1e-9`:

| samples K | allowance | vs. the true 1.91 | vs. the 11.0 cap |
|---|---|---|---|
| 340 | 4.20 | 2.20× | 0.38× |
| 1,000 | 2.78 | 1.45× | 0.25× |
| 3,400 | 2.22 | 1.16× | 0.20× |

So sampling is clearly worth doing — at 340 samples it is nearly three times
tighter than assuming the cap. But the gain decays slowly, because at these
sample counts the bound is dominated by the range term `3*R*L/K`, which does
not depend on what was observed and shrinks only as `1/K`. At `K = 340` it
contributes 1.93 of the 3.20.

An implementation should not expect to escape this by sampling harder. The
number of distinct heights a skeleton walk can ask about over a sampling window
caps `K` in the low thousands, and with it the allowance at roughly 1.4× the
truth. **Closing the rest is not a sampling problem**, and it is not a header
field either.

An earlier revision of this document said that a header committing to its own
cumulative difficulty would make the quantity exact and remove the uncle term.
That was wrong. Such a field is *data*: a miner can mine a header with valid
proof of work and write any cumulative total into it, because the rule binding
the field to its parent's value is enforced by nodes that validate the parent's
**body**, which the client in question does not have. Trusting it replaces a
proof with an assertion, and the assertion is worth less than the bound it would
replace:

```
  honest chain claims   1.91 x its trunk work   (measured mainnet uncle rate)
  attacker claims      11.00 x its trunk work   (uncleListLimit, fabricated)
  attacker needs        1.91 / 11 = 17.4%  of the honest chain's trunk hashpower
```

What closes it is **exhibiting the uncles** -- RSKIP-698 -- because an uncle
header carries its own proof of work, `unclesHash` binds the list to a block
whose proof of work covers it, and the uncle rules are checkable against the
trunk chain already in hand. That costs about 10.5 GB on mainnet today and no
field makes it cheaper, because work is proven by exhibiting it. Shrinking the
proof needs a different proof *system*, not another field.

Whichever shape is chosen, the property to test is that the ceiling exceeds the
honest chain's **uncle-inclusive** work at every spacing — not only at the one
that happens to be configured.

### What it costs, and how tight it is

The bound loosens roughly as `(1 + 1/divisor)^(N/2)` across a gap of `N`
blocks, so the interval trades tightness against request count; the uncle
allowance adds its own factor on top, and at reachable sample counts that
factor is the larger of the two.

Measured end to end, with 340 samples spread over 276,000 blocks above the
checkpoint at a divisor of 400: the ceiling sits about **3.9× the real work in
the sampled window**. That window is about 6% of the cumulative total, the rest
being exact, so against the figure a peer actually claims the slack is about
**17%**.

That is the honest number and it is not a small one: a peer can overstate its
work by a sixth and be judged plausible. It is still worth having, because the
claim an attacker needs to make is not 17% high but orders of magnitude high —
the gate exists to refuse a fabricated chain, not to referee a close race.

The samples come from one peer, so the first budget is requests rather than
bytes. **Each sampled height costs one message**: scattered heights cannot be
batched, since `GetBlockHeaders(start, count, skip)` walks a fixed stride and
the random draw is deliberately not one. Paying a message per header is the
price of unpredictability.

Over a window `W` above the checkpoint, with `K` random samples and a maximum
gap `N`, that is `max(K, W/N)` messages plus the skeleton walk. At 276,000
blocks, `K = 340` and `N = 768`: about **585 messages and 0.6 MB**, some
thirty-five seconds against rskj's limit of 1,000 messages per minute per peer.
Samples need headers only, never uncles — `uncleCount` is a header field.

Sampling the whole chain rather than a window would be about 34 minutes, which
is the reason the checkpoint exists.

### When to sample, and when to just download the window

An implementation should know that sampling is not always the cheaper move.

The alternative is to download every header **with its uncles** (RSKIP-698)
over the window and compute the cumulative difficulty exactly — no ceiling, no
allowance, no concentration argument. A header walk alone cannot: uncle
difficulty counts toward the total and uncle headers travel only in block
bodies, which is the gap RSKIP-698 closes.

A walk batches headers — 192 per message in both rskj and this implementation —
against the sampler's one. So:

```
  walk messages   = W / 192
  sample messages = max(K, W / N)
```

These meet at `W = K * 192`, which for `K = 340` is **65,280 blocks**, about
twenty-three days of chain. Below it the selection finds fewer available
heights than `K` and returns all of them: it stops being a sample, spends a
walk's worth of messages to retrieve a hundred and ninety-second of the data,
and ends with a loose bound where the walk ends with the exact number. **In
that regime sampling is strictly dominated** and an implementation should walk
instead.

**The condition is the capability, not only the window.** The walk yields the
exact total only from a peer that sends the uncles. From an `rsk/62` peer it
sums trunk difficulty alone, which is a *lower* bound on the chain's work —
worse than the gate's upper bound, not better, since it understates an honest
peer and so refuses it. Capabilities are negotiated per peer and known before a
status is acted on, so an implementation can decide this at run time and need
not wait for a network-wide upgrade:

```
  if peer supports RSKIP-698 and W < K * 192:
      walk the window with uncles, compute the total exactly, skip the gate
  else:
      sample
```

The gate then stops running exactly where it was dominated, against the peers
that can replace it, and keeps running everywhere else.

| window | days | sample msgs | walk msgs | walk bytes | |
|---|---|---|---|---|---|
| 40,000 | 14 | 208 | 209 | 96 MB | walk |
| 65,280 | 23 | 338 | 340 | 157 MB | walk |
| 100,000 | 35 | 349 | 521 | 240 MB | marginal |
| 276,000 | 96 | 513 | 1,438 | 663 MB | sample |

**Bytes never favour the walk**, by about a thousandfold — 0.6 MB against
663 MB at the last row, and 0.39 MB against 157 MB at the crossover. That
matters because the cost is paid **per candidate peer**: vetting is what a
client does to peers it does not yet trust, and it wants to do it to several.
Five peers cost 3 MB sampled and 3.3 GB walked, the latter being larger than
the state download the sync exists to perform.

The corollary is worth stating because it is the opposite of the intuition:
with a checkpoint refreshed each release, a client on a current build sits
below the crossover. **The gate matters least when the checkpoint is fresh and
most for clients on stale builds.**

### Vetting several peers at once

Each peer gets its **own independent draw**.

- The heights do not mean the same thing. A sampler is built from *that peer's*
  head and *that peer's* skeleton; two peers at different heads have different
  windows, and if they are on different chains — the case being vetted for — a
  given height is a different block for each.
- **Sybil cost must scale with identities.** Requests go out concurrently, so
  an attacker running `N` identities sees the draw as soon as the first is
  queried. A shared draw lets one set of mined headers answer for all `N`:
  `O(K)` rather than `O(KN)`.
- **It must not become a vote.** Asking every peer the same heights and
  comparing answers is majority voting among peers, which is what sybils are
  cheap against. Each peer is judged against arithmetic, never against other
  peers.

Verified **answers** may be shared: a header is bound by its hash, so one that
two skeletons agree on needs its proof of work checked once. **Positions** may
not.

The draw must come from a CSPRNG seeded from the OS, never from anything a peer
can see, and must not be reused across sessions with the same peer — one that
failed once would otherwise learn where to be honest next time.

### Two separable decisions

Checking that a peer is on the checkpointed chain — asking for the header at
the checkpoint height and comparing its hash — and bounding the work it claims
are **different decisions, and should be separately configurable**.

Shipping a checkpoint *hash* is a statement about which chain is canonical: it
asks whoever builds the node to choose a fork, which is a governance act.
Bounding the work makes no such statement — it says only that a claim exceeds
what any chain could carry. An operator may reasonably want the second without
the first.

Without the hash check, the work below the checkpoint is credited to any peer,
including one on a fork that diverged below it and never did that work. On
mainnet that is over 99% of the cumulative total, so the bound discriminates
only within the sampled window.

### What it does not do

It does not say *which* chain a peer is on, only how much work it may claim. A
peer serving a valid minority chain within the ceiling passes. It is a filter
on who is worth talking to, **not a substitute for the header walk**, which
remains the thing that establishes the checkpoint is on a chain this node
accepts.

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

A client SHOULD bound the claimed work before committing to a long walk — see
"Bounding the claim before the walk" — and SHOULD treat a peer whose
difficulties fail the pairwise check as having offered an unusable snapshot
rather than as merely slow.

Note what the bound does **not** replace. It establishes that a claim is
arithmetically possible; the header walk establishes that the checkpoint sits
on a chain this node accepts. A client that bounds the claim and skips the walk
has checked that a lie was plausible, not that it was absent.
