---
rskip: 517
title: Block time-centric difficulty adjustment with uncle threshold
description: Modifies difficulty adjustment algorithm to prioritize block time over uncle rate, enhancing stability and security.
status: Draft
purpose: Sca,Sec
author: PDG (@patogallaiovlabs)
layer: Core
complexity: 2
created: 2025-05-29
---

# Block time-centric difficulty adjustment with uncle threshold

| RSKIP          | 517                                                           |
| :------------ |:-------------|
| **Title**      | Block time-centric difficulty adjustment with uncle threshold |
| **Created**    | 29-MAY-2025                                                   |
| **Author**     | Patricio Gallardo                                             |
| **Purpose**    | Sca,Sec                                                           |
| **Layer**      | Core                                                          |
| **Complexity** | 2                                                             |
| **Status**     | Draft                                                         |

## Abstract

This RSKIP proposes a change of the difficulty adjustment algorithm to primarily consider block time over uncle rate. The primary goal is to reduce the network's average block time by lowering the mining difficulty. However, reducing the difficulty increases the risk of uncle block generation due to current mining pool behaviors related to block template refresh intervals. To address this, the proposal introduces a threshold-based mechanism that allows difficulty to decrease without being overly penalized by temporary increases in uncle rate, thereby improving network stability. The current method also relies on per-block signals, which often misrepresent broader network trends due to the statistical nature of block time distribution. This proposal introduces a window-based averaging mechanism that better estimates the underlying block time. This approach yields a more stable and accurate approximation of network performance.

## Motivation

Currently, difficulty in Rootstock is adjusted based on both block time and uncle rate. However, due to how mining pools handle their block template refresh, reducing the difficulty typically results in an increased rate of uncle blocks. This creates a feedback loop: as the number of uncles rises, it affects the difficulty calculation, which then counteracts the intended difficulty reduction.

Since the ultimate objective is to reduce the average block time (as discussed in RSKIP491²), we propose a mechanism where block time becomes the primary driver of difficulty adjustments. Uncles will still be monitored, but they will only influence the difficulty when their rate crosses a defined threshold, signaling potential instability. This approach avoids the overreaction to uncle spikes and allows a controlled reduction in difficulty to achieve faster block production.

This design complements RSKIP77¹, which proposed smoothing mechanisms using sliding windows but did not decouple uncles from the difficulty logic. It also aligns with statistical reasoning: while individual block times follow an exponential distribution and exhibit asymmetric variability, aggregating over a window shifts the distribution closer to normal due to the Central Limit Theorem⁴. This results in more accurate estimates of the true mean and helps avoid overcorrection based on short-term deviations.

## Specification

### Parameters and definitions

* $B$ = last N blocks
* $B_i$ = block in position $i$ of $B$, where $B_1$ is the first block in the set and $B_N$ is the last block.
* $T_{B_i}$ = block time for block $i$
* $U_{B_i}$ = uncles referenced by block $i$
* $D_{B_i}$ = difficulty of block $i$
* $T$ = target time between blocks in seconds (currently is set to 14)
* $C$ = uncle threshold (e.g., 0.5 means threshold supports up to 1 uncle every 2 blocks)
* $\alpha$ = base difficulty change factor (currently is set to 0.0025)

### Background

Each time a new block is mined, the difficulty for the next block is recalculated using the most recent block's difficulty and a scaling factor $F$. This adjustment helps maintain a consistent block interval by increasing or decreasing difficulty in response to recent network conditions. The formula used is:

$$
D_{B_{N+1}} = D_{B_{N}} \cdot (1 + F)
$$

The current difficulty adjustment factor can be defined as:

$$
F = \begin{cases}
\alpha & \text{if }  T\cdot (1+U_{B_N}) \geq T_{B_N} \\
-\alpha & \text{if } T\cdot (1+U_{B_N}) < T_{B_N} \\
\end{cases}
$$

RSKIP77 proposes an alternative method that uses an N-block (N=32) sliding window:

$$
F = \begin{cases}
\alpha & \text{if } \left(\sum_{i=1}^{N} 1 + U_{B_i}\right)\cdot T \geq T_{B_N} - T_{B_1} \\
-\alpha & \text{if } \left(\sum_{i=1}^{N} 1 + U_{B_i}\right)\cdot T < T_{B_N} - T_{B_1} \\
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

The new difficulty adjustment introduces a conditional factor $F$ that adapts based on both the uncle rate and deviation from the target block time. This mechanism ensures that uncle blocks impact difficulty when their rate exceeds a defined threshold, preserving network stability while allowing controlled block time reductions. The adjustment factor is defined as:

$$
F = \begin{cases}
\alpha & \text{if } R > C \\
\alpha & \text{if } C \leq R \land A > T \cdot 1.10 \\
-\alpha & \text{if } C  \leq R  \land A < T \cdot 0.90 \\
0 & \text{otherwise}
\end{cases}
$$

The threshold $C$ should be defined based on empirical data from stress testing the network. For example, a value of 1 may reflect the typical uncle frequency beyond which network stability is degraded. Similarly, $T$ should be selected in conjunction with $C$ to represent the maximum combination the network can handle without compromising stability.

Given that the network currently³ produces main blocks every 24 seconds with an uncle rate of 0.5, we suggest the following conservative parameter values for the new calculation:

* $\alpha \small\text{(base difficulty change factor)} = 0.0025$
* $T \small\text{(target block time) = 20 (with a tolerance range of ±10%, i.e., [18, 22])}$
* $C \small\text{(uncle threshold)} = 0.7$
* $N \small\text{(window size / difficulty adjustment frequency)} = 30 (blocks)$ 

## Rationale

Uncles are an indirect signal of network congestion or miner behavior, not a direct reflection of time-based difficulty needs. Currently, difficulty is highly sensitive to uncles, which undermines attempts to reduce block time. This proposal decouples the automatic inclusion of uncles in every difficulty update, instead applying a threshold so they influence the algorithm only when truly abnormal.

This preserves network security by ensuring uncle production doesn't spiral out of control, while also enabling more accurate and less volatile difficulty changes.  The block time becomes the main feedback signal, aligning the difficulty with actual network performance and the long-term goal of reducing average block time.

To further strengthen this approach, we incorporate statistical reasoning into the choice of difficulty update inputs. The mining process follows an exponential distribution, where each block time represents a random sample. Attempting to adjust difficulty based on individual block times introduces error, since the exponential distribution is not symmetric and its mean is not centered. However, when summing multiple exponential variables—as done with a sliding window—their distribution converges toward a normal distribution by the Central Limit Theorem⁴. This aggregated approach yields a more reliable estimate of the true mean block time, improving the accuracy and stability of difficulty adjustments.

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

\[4] Central limit theorem: [Wikipedia entry](https://en.wikipedia.org/wiki/Central_limit_theorem)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
