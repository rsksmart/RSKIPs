---

rskip: pull\_request\_number\_here
title: Block time-centric difficulty adjustment with uncle threshold
created: 14-MAY-2025
author: PDG (@patogallaiovlabs)
purpose: Sec
layer: Core
complexity: 2
status: Draft
description: Modifies difficulty adjustment algorithm to prioritize block time over uncle rate, enhancing stability and security.
---------------------------------------------------------------------------------------------------------------------------------

| RSKIP          | pull\_request\_number\_here                                   |
| :------------- | :------------------------------------------------------------ |
| **Title**      | Block time-centric difficulty adjustment with uncle threshold |
| **Created**    | 14-MAY-2025                                                   |
| **Author**     | Patricio Gallardo                                             |
| **Purpose**    | Sec                                                           |
| **Layer**      | Core                                                          |
| **Complexity** | 2                                                             |
| **Status**     | Draft                                                         |

## Abstract

This RSKIP proposes a revision of the RSK difficulty adjustment algorithm to primarily consider block time over uncle rate. The objective is to improve network stability by minimizing volatility in block production and difficulty shifts due to temporary uncle rate fluctuations. It also introduces a threshold mechanism where uncle rate only influences difficulty if it exceeds a specified value.

## Motivation

The existing RSK difficulty algorithm considers both block time and uncle rate, leading to high sensitivity to uncle production. Due to the inherent randomness in block production and delayed uncle inclusion, this dual consideration can cause instability and overreaction in difficulty adjustments. The new method simplifies this mechanism by focusing on block time and limiting uncle rate influence to situations where it exceeds a defined threshold.

This design supports initiatives such as those in [RSKIP491](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP491.md), which target lower block times, and complements previous smoothing proposals like [RSKIP77](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP77.md) that use sliding windows to average difficulty.

## Specification

### Parameters and definitions

* $B$ = last N blocks
* $B_i$ = block in position $i$ of $B$
* $T_{B_i}$ = block time for block $i$
* $U_{B_i}$ = uncles referenced by block $i$
* $D_{B_i}$ = difficulty of block $i$
* $T$ = target time between blocks
* $C$ = uncle threshold
* $\alpha$ = base difficulty change factor (e.g. 0.05)

### Average block time

$$
A = \frac{\sum_{i=1}^{N} T_{B_i}}{N}
$$

### Uncle rate

$$
R = \frac{\sum_{i=1}^{N} U_{B_i}}{N}
$$

### Difficulty adjustment factor

$$
F = \begin{cases}
\alpha & \text{if } R > C \\
\alpha & \text{if } R \leq C \land T > A \\
-\alpha & \text{if } R \leq C \land T < A \\
\end{cases}
$$

### Updated difficulty

$$
D_{\text{new}} = D_{\text{last}} \cdot (1 + F)
$$

## Rationale

Uncles are an indirect signal of network congestion or miner behavior, not a direct reflection of time-based difficulty needs. Prioritizing block time ensures a more deterministic and stable adjustment. The uncle threshold offers a backstop mechanism, engaging only when uncle rate signifies possible instability or miner misalignment.

Compared to RSKIP77’s averaging approach across a sliding window of 32 blocks with all uncles considered, this method is more robust to unintentional spikes and optimizes around actual block production pace.

## Backward compatibility

This is a hard-forking change to the consensus rules. All full nodes must upgrade to remain on the canonical chain. SPV clients are not affected. Coordination with miner software and monitoring tools is recommended to adjust expectations around difficulty behavior.

## Test cases

To be developed alongside the implementation. Test cases will include:

* Validation of block time calculations.
* Uncle rate threshold exceedance behavior.
* Difficulty progression under various block and uncle production patterns.

## References

\[1] RSKIP77 - Smoother Difficulty Adjustment
\[2] RSKIP491 - Reduce Target Difficulty to Lower Average Block Time to 10s
\[3] Rootstock Blog: [Leveraging Bitcoin's Security](https://rootstock.io/blog/leveraging-bitcoins-security-exploring-the-dynamics-of-merged-mining/)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
