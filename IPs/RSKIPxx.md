---
rskip: TBD
title: StableMinGasPrice
description:
status: Draft
purpose: Usa
author: @rmoreliovlabs
layer: Core
complexity: 2
created: 2024-06
---
# EthSwap


|RSKIP          | TBD |
| :------------ |:-------------|
|**Title**      |StableMinGasPrice|
|**Created**    |JUNE-2024 |
|**Author**     |@rmoreliovlabs |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes a feature that allows RSK miners to configure the minimum gas price (MinGasPrice) in fiat currency instead of the current cryptocurrency (WEI) linked to Bitcoin.

## Motivation

In the current RSK network, miners have the ability to set the minimum gas price through the `miner.minGasPrice` configuration, which is denominated in WEI. This parameter is essentially linked to the Bitcoin price, causing the gas price to vary in correlation with Bitcoin's value. This behaviour creates uncertainty in transaction costs, often leading to increased costs when Bitcoin appreatiates. This unpredictability can negatively impact user experience, as transaction fees become less stable.

This can be solved adding a new feature that allows miners to specify and configure the minimum gas price in fiat currency. By decoupling gas prices from Bitcoin's market fluctuations, this property  aims to stabilize transaction costs in terms of fiat value, thereby improving predictability. This approach not only protects users from the volatility of cryptocurrency markets but also provides a better user experiencie.

