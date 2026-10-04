---
rskip: 699
title: Add the cumulativeDifficulty field to the Block header extension
status: Draft
purpose: Usa
author: SDL
layer: Core
complexity: 1
requires: 144, 194, 351, 535
created: 2026-10-04
---

# Add the cumulativeDifficulty field to the Block header extension

## Abstract

This RSKIP adds a field `cumulativeDifficulty` to the Rootstock block header
extension, following the rules of [RSKIP 194](./RSKIP194.md) and
[RSKIP 351](./RSKIP351.md), and defines block header version 3.

The field holds the total difficulty of the chain ending at the block,
including the difficulty of every referenced uncle. It makes the chain's
accumulated work readable from a single header, which today it is not.

## Motivation

Rootstock counts uncle difficulty in cumulative work. A block contributes

```
block.difficulty + sum(uncle.difficulty for uncle in block.uncles)
```

This differs from Ethereum, where only the trunk block's own difficulty is
counted, and the difference has a consequence that is easy to miss: **the
cumulative difficulty of a Rootstock chain cannot be computed from its block
headers.** The trunk headers carry `uncleCount` but not the uncle headers, and
an uncle's difficulty is not derivable from the trunk header that references
it. To add up the work, a node must obtain the uncle headers, which exist
nowhere but inside block bodies.

This is the whole of the problem, and it shows up in three places.

### Header-only sync cannot choose a chain

A node that downloads only headers cannot compute the cumulative difficulty of
what it downloaded, and therefore cannot tell which of two chains has more
work. Choosing the heaviest chain is not an optimisation a client may skip; it
is what makes the client a participant in Nakamoto consensus rather than a
party trusting whoever it connected to. Today a Rootstock client that wants
that answer must download block bodies, or a parallel uncle-header stream, for
the entire history.

Measured against this chain at #9,276,564, the uncle headers a node must hold
to add up work amount to **10.5 GB**, against **10.1 GB** for the trunk headers
themselves. Work accounting therefore costs slightly more than doubling a
header download. The same figure as a single field in each header is
**130 MB** -- about **80x** less.

### Light clients and zero-knowledge proofs

[RSKIP 535](./RSKIP535.md) adds `baseEvent` so that a chain's work backing a
peg-out event can be proven in zero knowledge cheaply. Proving accumulated work
runs into the same obstruction: the prover must carry uncle headers to sum
difficulty, which multiplies the circuit's input. A committed cumulative
difficulty reduces "how much work backs this block" to reading one field.

### Peer claims are bounded rather than checked

A client that samples headers rather than walking all of them -- as snapshot
sync must, before it has anything to walk -- has to bound a peer's claimed
total difficulty rather than compute it. Because uncle work is not visible in
the headers it sampled, that bound must include an allowance for uncles it did
not see, and an allowance sound against the consensus limit of ten uncles per
block is an order of magnitude wide.

This RSKIP does not eliminate that bound; see
[Security considerations](#security-considerations). It does remove the uncle
term from it for a client that walks the headers it is judging.

## Specification

A new block header version 3 is defined. All blocks after the network upgrade
must use this version number.

As in versions 1 and 2, there are two separate serialization methods: one for
the transfer of block header information, and one for the computation of the
block hash. This RSKIP changes neither the compressed header nor the fields the
PowHSM parses.

One field is added to the Block Header Extension:

- For version 3:
  ```
  blockHeaderExtensionEncoded = RLP(logsBloom, txExecutionSublistsEdges,
                                    baseEvent, cumulativeDifficulty)
  blockHeaderExtensionHash = Keccak256(RLP(
    Keccak256(logsBloom), txExecutionSublistsEdges,
    baseEvent, cumulativeDifficulty))
  ```

As in version 2, a field belonging to an inactive feature is left empty rather
than omitted, so that the list length does not vary with the active feature
set.

### Value

`cumulativeDifficulty` is encoded as an RLP byte string holding the
unsigned big-endian representation of

```
cumulativeDifficulty(genesis) = genesis.difficulty

cumulativeDifficulty(B) = cumulativeDifficulty(B.parent)
                        + B.difficulty
                        + sum(u.difficulty for u in B.uncles)
```

with no leading zero bytes.

The encoding is **unsigned**, departing from the signed two's-complement form
that `java.math.BigInteger.toByteArray()` produces and that appears elsewhere
in Rootstock's header RLP. The current value has its high bit set, so the two
differ today: a signed encoding spends a thirteenth byte on a sign that the
quantity, being a count of work, can never use. An implementation must reject a
`cumulativeDifficulty` whose first byte is zero.

### Consensus rule

For blocks of version 3 and above, a block is invalid unless its
`cumulativeDifficulty` equals the value defined above. The rule is checked
against the parent's field, so it is O(1) per block and costs a full node
nothing it was not already computing.

### Activation

At the network upgrade, the field of the first version 3 block is computed from
its parent's cumulative difficulty as the node has it, which every node already
maintains outside the header. Nodes must agree on that value at the activation
height; this is the same assumption that makes the existing fork-choice rule
work, and it is worth an explicit statement in the upgrade's release notes
rather than being left implicit.

## Rationale

### Why the extension, and not the header

The header proper is parsed by the PowHSM firmware. [RSKIP 194](./RSKIP194.md)
exists precisely so that new fields can be added without a firmware upgrade,
and [RSKIP 351](./RSKIP351.md) and [RSKIP 535](./RSKIP535.md) have each used
that room. This follows them. The compressed header is unchanged, the block
hash preimage is unchanged in shape, the PowHSMs need no new firmware, and the
merge-mining path is untouched.

The cost is that the PowHSM cannot read the field. It accumulates difficulty
itself and will continue to; this RSKIP does not change what it does.

### Size

Measured on this chain, a serialized extended header averages **1,145 bytes**
over the last 76,000 blocks, with a median of 1,150.

The field is 13 bytes of value plus one RLP prefix byte: **14 bytes**, or
**+1.22%**.

It stays 14 bytes for the foreseeable life of the chain. Cumulative difficulty
is at 96 bits today; at the current rate of accumulation it reaches 98 bits in
ten years and 101 bits in a hundred, so the encoding does not widen.

For a light client, the comparison is better than +1.22%, because such a client
takes the *compressed* extension -- `Keccak256(logsBloom)` in place of the
256-byte filter, which is what RSKIP 351 provides the hash form for. That
header is about 921 bytes, so the field costs **+1.5%** there, and replaces the
~1,257 bytes per block of uncle headers the client needed to do the same job.
Work accounting goes from roughly doubling a light client's download to adding
one and a half percent to it.

### Why not derive it from a separate structure

A side structure -- a per-block cumulative difficulty served by a new wire
message, or an accumulator committed once per epoch -- would avoid touching the
header. It would also be unverifiable on its own: the recipient would have no
way to tell a correct side structure from a fabricated one without the uncle
headers it was trying to avoid. Putting the value where proof-of-work already
commits to it is what makes it worth anything.

### Why unsigned

A signed encoding costs a byte whenever the high bit is set, which is the
present state of the chain and will remain so for most of any byte width's
range. More importantly, the signed convention has already caused one
interoperability failure in practice: an implementation that wrote the
unsigned form and a peer that parsed the signed one disagreed about a total
difficulty whose high bit had just been set, and the peer rejected every status
message. A new field is the opportunity to not inherit that, and the rejection
rule above makes the two forms distinguishable rather than silently
interchangeable.

## Backwards compatibility

- This change requires a network upgrade. All full nodes must be upgraded.
- RPC responses for block queries gain a field. `eth_getBlockByNumber` already
  reports `totalDifficulty`, computed by the node; it will now be reporting a
  value the header states. Clients need no change.
- PowHSM firmware needs no upgrade, and the powpeg nodes do not need to decode
  the field.
- Blocks before the activation height are version 2 or lower and are unchanged.
  A node syncing history computes their cumulative difficulty as it does today.

## Security considerations

**What the field does not do.** It does not let a client conclude that work was
performed. A header's proof of work attests to that header's own difficulty and
to the contents of its extension, including this field -- but a miner can mine
a valid header stating any `cumulativeDifficulty` it likes, because the
consensus rule binding the field to the parent's is enforced by nodes that
validate the parent, not by the proof of work.

A client that walks every header from a point it trusts is therefore in a
strong position: it can read the field instead of summing, and needs no uncle
headers. A client that samples headers is not, and must still bound the work
between samples by what the difficulty retarget rule allows. The gain there is
narrower but real: the bound no longer has to carry an allowance for unseen
uncle difficulty, because the field at each sampled header states the
cumulative total exactly, and the quantity being bounded between samples is the
*increment*, which remains governed by the retarget rule as before.

**Monotonicity.** Implementations should check that `cumulativeDifficulty`
strictly increases from parent to child, and reject a header where it does not,
rather than relying on the equality check alone. The two are equivalent for a
correct chain and the redundant check costs nothing.

**No new denial of service.** The field is fixed-width in practice, bounded in
the specification by the RLP length prefix, and validated by one addition
against a value the node already holds.

_Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)._
