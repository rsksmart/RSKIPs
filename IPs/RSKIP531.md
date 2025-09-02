---
rskip: 531
title: Mitigation of Fee-Withholding Incentive in REMASC
description: Proposes a modification to REMASC to prevent fee-withholding strategies by introducing immediate partial fee distribution.
status: Draft
purpose: Sec
author: PDG (@patogallaiovlabs)
layer: Core
complexity: 2
created: 2025-09-01
---

# Mitigation of Fee-Withholding Incentive in REMASC

| RSKIP          | 531                       |
| :------------ |:-------------|
| **Title**      | Mitigation of Fee-Withholding Incentive in REMASC |
| **Created**    | 01-SEP-2025                                       |
| **Author**     | Patricio Gallardo                         |
| **Purpose**    | Sec                                         |
| **Layer**      | Core                                              |
| **Complexity** | 2                                                 |
| **Status**     | Draft                                             |

## Abstract

This RSKIP proposes modifying the REMASC reward scheme to prevent fee-withholding strategies that degrade network performance. Strategic miners may exclude transaction fees to maximize their REMASC rewards, but this behavior reduces network throughput, increases confirmation delays, and creates congestion. We propose a defense mechanism: immediate partial fee distribution. A small fraction (e.g., 3%) of transaction fees would be instantly awarded to the current block producer, eliminating the incentive for fee withholding while preserving REMASC's security benefits.

## Motivation

REMASC introduces a delayed miner reward system to incentivize long-term honest behavior. However, its decoupling of fee inclusion and reward attribution creates an incentive for strategic miners to withhold transaction fees. This fee-withholding strategy, while potentially profitable for individual miners, actively degrades the network in several ways:

* **Increased Confirmation Delays**: Strategic miners delay transaction inclusion to future blocks where they can profit from the fees, slowing down confirmations
* **Economic Inefficiency**: Higher fees actually incentivize miners to delay inclusion further, creating a perverse incentive where users pay more for slower service
* **Uneven Reward Distribution**: Strategic fee withholding creates an unfair advantage, favoring miners who adopt this behavior over those who don't
* **No Penalization**: Excluded transactions simply roll over to future blocks, meaning there's no cost to miners for withholding fees

We demonstrate both theoretically and empirically that this strategy yields an advantage for medium-hashrate strategic miners while degrading overall network performance \[3]. Addressing this vulnerability is critical to maintaining network efficiency and user experience.

## Specification

Introduce a protocol change where a configurable fraction $\alpha \in [0,1]$ of the transaction fees in each block is immediately credited to the miner of that block. The remaining $1 - \alpha$ continues to accumulate within REMASC and is distributed according to the existing delayed reward logic.

* Modify REMASC contract to:

  * On block inclusion, distribute $\alpha \cdot block.fees$ to `block.coinbase`
  * Accumulate $(1 - \alpha) \cdot block.fees$ as usual
* Initial default value: $\alpha = 0.03$

## Rationale

Simulations \[3] confirm that even low $\alpha$ (e.g., 0.03) removes the advantage or turns it negative, making the fee-withholding strategy irrational. By providing immediate partial fee distribution, this change:

* **Eliminates Network Degradation**: Removes the incentive to withhold fees, ensuring consistent network performance
* **Maintains REMASC Benefits**: Preserves the delayed reward incentives for long-term security
* **Improves User Experience**: Ensures predictable transaction confirmation times and fee structures
* **Creates Alignment**: Aligns miner incentives with network health and user satisfaction

This modification transforms fee-withholding from a profitable but network-degrading strategy into an unprofitable one, while maintaining REMASC's core security objectives.

## Backwards Compatibility

This is a consensus-breaking change and requires a hard fork. All nodes must upgrade. Light clients are unaffected.

## References

\[1] REMASC Documentation – [https://github.com/rsksmart/remasc](https://github.com/rsksmart/remasc)

\[2] RSKIP15 – Security Bonding via Delayed Rewards [https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP15.md](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP15.md)

\[3] Mining Reward Simulation Tools – [https://github.com/patogallaiovlabs/sim-rewards](https://github.com/patogallaiovlabs/sim-rewards)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
