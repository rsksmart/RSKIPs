---
rskip: 77
title: Smoother Difficulty adjustment
description: 
status: Draft
purpose: Sca, Fair
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016
---
#  **Smoother Difficulty adjustment**  

| RSKIP          | 77                             |
| :------------- | :----------------------------- |
| **Title**      | Smoother Difficulty adjustment |
| **Created**    | 2016                           |
| **Author**     | SDL                            |
| **Purpose**    | Sca, Fair                      |
| **Layer**      | Core                           |
| **Complexity** | 2                              |
| **Status**     | Draft                          |

# Abstract

This RSKIP proposes that a new difficulty adjustment algorithm. The algorithm was originally proposed in RSKIP15, but left outside the final feature set merged. 

# Motivation

The current difficulty adjustment algorithm only takes into account the parent block time and the number of uncles. As uncle inclusion is not immediate, this creates a dynamic feedback system with a delay 2-6 blocks delay. This can generate undesired oscillations in difficulty.  As the current difficulty adjustment step is +-2048, 6 block oscillations would be "on average" +-314 (0.29%). Expected variance could lead to +=3% deviations over the course of a day. These oscillations should not interfere with normal mining process, but anyway could be avoided by extending the interval where difficulty measurements are made. Also by extending this interval we reduce the chance of difficulty manipulation by miners.

## Specification

The difficulty adjustment should be computed as the amount of accumulated work of all blocks (main and uncle blocks) created in a block interval (a number of main chain blocks) over the time interval (difference of timestamps between first and last block). The block interval is 32 blocks, to reduce variance but allow for adjustments in less than 5 minutes (for upward adjustments). The adjustment must be performed at every block, using 32-block overlapping windows. Because uncles can be included as late as 10 blocks after the sibling block, the block interval will consider all uncles references, no matter they compete to a sibling block in the current interval or in the previous interval. This introduces an error factor in the formula, but this can be tolerated to reduce the complexity of the algorithm and also to avoid computing the new difficulty over a window that is 10 blocks old. In the following diagram, two includes are included in the counting, even if the first uncle parent is outside the window of interest. 



![](RSKIP77\image2.png)

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


