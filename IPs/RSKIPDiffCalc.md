---
rskip: XXX
title: Block time-centric difficulty adjustment with uncle threshold
description: Modifies difficulty adjustment algorithm to prioritize block time over uncle rate, enhancing stability and security.
status: Draft
purpose: Sec
author: PDG (@patogallaiovlabs)
layer: Core
complexity: 2
created: 2025-05-15
---

# Block time-centric difficulty adjustment with uncle threshold

| RSKIP          | XXX                                                           |
| :------------ |:-------------|
| **Title**      | Block time-centric difficulty adjustment with uncle threshold |
| **Created**    | 14-MAY-2025                                                   |
| **Author**     | Patricio Gallardo                                             |
| **Purpose**    | Sec                                                           |
| **Layer**      | Core                                                          |
| **Complexity** | 2                                                             |
| **Status**     | Draft                                                         |

## Abstract

This RSKIP proposes a change of the difficulty adjustment algorithm to primarily consider block time over uncle rate. The primary goal is to reduce the network's average block time by lowering the mining difficulty. However, reducing the difficulty increases the risk of uncle block generation due to current mining pool behaviors related to block template refresh intervals. To address this, the proposal introduces a threshold-based mechanism that allows difficulty to decrease without being overly penalized by temporary increases in uncle rate, thereby improving network stability.

## Motivation

Currently, difficulty in Rootstock is adjusted based on both block time and uncle rate. However, due to how mining pools handle their block template refresh, reducing the difficulty typically results in an increased rate of uncle blocks. This creates a feedback loop: as the number of uncles rises, it affects the difficulty calculation, which then counteracts the intended difficulty reduction.

Since the ultimate objective is to reduce the average block time (as discussed in RSKIP491²), we propose a mechanism where block time becomes the primary driver of difficulty adjustments. Uncles will still be monitored, but they will only influence the difficulty when their rate crosses a defined threshold, signaling potential instability. This approach avoids the overreaction to uncle spikes and allows a controlled reduction in difficulty to achieve faster block production.

This design complements  RSKIP77¹, which proposed smoothing mechanisms using sliding windows but did not decouple uncles from the difficulty logic.

## Specification

### Parameters and definitions

* $B$ = last N blocks
* $B_i$ = block in position $i$ of $B$
* $T_{B_i}$ = block time for block $i$
* $U_{B_i}$ = uncles referenced by block $i$
* $D_{B_i}$ = difficulty of block $i$
* $T$ = target time between blocks in seconds (currently is set to 14)
* $C$ = uncle threshold (e.g., 0.5 means threshold supports up to 1 uncle every 2 blocks)
* $\alpha$ = base difficulty change factor (currently is set to 0.0025)

### Background

Each time a new block is mined, the difficulty for the next block is recalculated using the most recent block's difficulty and a scaling factor $F$. This adjustment helps maintain a consistent block interval by increasing or decreasing difficulty in response to recent network conditions. The formula used is:

$$
D_{B_{n+1}} = D_{B_{n}} \cdot (1 + F)
$$

The current difficulty adjustment factor can be defined as:

$$
F = \begin{cases}
\alpha & \text{if }  T\cdot (1+U_{B_n}) \leq T_{B_n} \\
-\alpha & \text{if } T\cdot (1+U_{B_n}) > T_{B_n} \\
\end{cases}
$$

RSKIP77 proposes an alternative method that uses an N-block (N=32) sliding window:

$$
F = \begin{cases}
\alpha & \text{if } \left(\sum_{i=1}^{N} 1 + U_{B_i}\right)\cdot T \leq T_{n} - T_{1} \\
-\alpha & \text{if } \left(\sum_{i=1}^{N} 1 + U_{B_i}\right)\cdot T > T_{n} - T_{1} \\
\end{cases}
$$

While RSKIP77¹ smooths difficulty oscillations by broadening the analysis window, it still incorporates uncle blocks directly into the calculation. This proposal takes a step further by separating the treatment of uncles and blocks, introducing a threshold mechanism.

### New difficulty calculation

We define the average block time $A$ as:

$$
A = \frac{\sum_{i=1}^{N} T_{B_i}}{N}
$$

and the uncle rate $R$ as:

$$
R = \frac{\sum_{i=1}^{N} U_{B_i}}{N}
$$

The new difficulty adjustment introduces a conditional factor $F$ that adapts based on both the uncle rate and deviation from the target block time. This mechanism ensures that uncle blocks only impact difficulty when their rate exceeds a defined threshold, preserving network stability while allowing controlled block time reductions. The adjustment factor is defined as:

$$
F = \begin{cases}
\alpha & \text{if } R > C \\
\alpha & \text{if } R \leq C \land T > A \\
-\alpha & \text{if } R \leq C \land T < A \\
\end{cases}
$$

The threshold $C$ should be defined based on empirical data from stress testing the network. For example, a value of 1 may reflect the typical uncle frequency beyond which network stability is degraded. Similarly, $T$ should be selected in conjunction with $C$ to represent the maximum combination the network can handle without compromising stability.

Since currently the network is generating main blocks every 24 seconds with an uncle rate of 0.5, we suggest as conservative values for the new calculation the parameters: $\alpha = 0.0025$, $T = 20$ and $C = 0.7$.

## Rationale

Uncles are an indirect signal of network congestion or miner behavior, not a direct reflection of time-based difficulty needs. Currently, difficulty is highly sensitive to uncles, which undermines attempts to reduce block time. This proposal decouples the automatic inclusion of uncles in every difficulty update, instead applying a threshold so they influence the algorithm only when truly abnormal.

This preserves network security by ensuring uncle production doesn't spiral out of control, while also enabling more accurate and less volatile difficulty changes. The block time becomes the main feedback signal, aligning the difficulty with actual network performance and the long-term goal of reducing average block time.

## Backward compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated.

## Test cases

To be developed alongside the implementation. Test cases will include:

* Validation of block time calculations.
* Uncle rate threshold exceedance behavior.
* Difficulty progression under various block and uncle production patterns.

## References

\[1] [RSKIP77 - Smoother difficulty adjustment](https://ips.rootstock.io/IPs/RSKIP77.html)

\[2] [RSKIP491 - Reduce target difficulty to lower average block time to 10s](https://ips.rootstock.io/IPs/RSKIP491.html)

\[3] Rootstock Blog: [Leveraging Bitcoin's Security](https://rootstock.io/blog/leveraging-bitcoins-security-exploring-the-dynamics-of-merged-mining/)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
