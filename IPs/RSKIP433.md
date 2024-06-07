---
rskip: 433
title: Stable Minimum Gas Price
description:
status: Draft
purpose: Usa
author: RM
layer: Core
complexity: 2
created: 2024-06
---
# Stable Minimum Gas Price


|RSKIP          | 433 |
| :------------ |:-------------|
|**Title**      |Stable Minimum Gas Price|
|**Created**    |JUNE-2024 |
|**Author**     |RM |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes a feature that allows RSK miners to configure the minimum gas price (MinGasPrice) in fiat currency instead of the current cryptocurrency (WEI) linked to Bitcoin.

## Motivation

In the current RSK network, miners have the ability to set the minimum gas price through the `miner.minGasPrice` configuration, which is denominated in WEI. This parameter is essentially linked to the Bitcoin price, causing the gas price to vary in correlation with Bitcoin's value. This behaviour creates uncertainty in transaction costs, often leading to increased costs when Bitcoin prices go up. This unpredictability can negatively impact user experience, as transaction fees become less stable.

This can be solved adding a new feature that allows miners to specify and configure the minimum gas price in fiat currency. By decoupling gas prices from Bitcoin's market fluctuations, this property  aims to stabilize transaction costs in terms of fiat value, thereby improving predictability. This approach not only protects users from the volatility of cryptocurrency markets but also provides a better user experiencie.

## Specification

We could introduce a new configuration property, `stableMinGasPrice` to specify the minimum gas price in fiat currency. Key elements of this configuration include:
1. **Enabled Flag:** Adding the `enabled` flag to disable the stable gas price feature. If this value is false, then the system should behave as it currently does, using the `miner.minGasPrice` configuration.
2. **source.method:** Defines the method to retrieve the Bitcoin price. It can be one of these 2 methods: 
    * `HTTP_GET`: Simple HTTP Get request to fetch the price.
    * `ETH_CALL`: Call to a Smart Contract that returns the price.
3. **Minimum Stable Gas Price:** `minStableGasPrice` to specify the minimum gas price in the chosen fiat currency.
4. **Refresh Rate:** `refreshRate` to determine how often the Bitcoin price should be refreshed.

It should look something like this:
```
miner {
    # The default gas price
    minGasPrice = 4265280000000

    stableGasPrice {
        enabled = false
        minStableGasPrice = 4265280000000 # 0.00000426528 USD per gas unit
        refreshRate = 360
        method = HTTP_GET || ETH_CALL
        
        #     Examples:
        #     method = "HTTP-GET"
        #     params {
        #         url = "https://domain.info/ticker"
        #         jsonPath = "/USD/buy"
        #         timeout = 2 seconds
        #     }
        #     
        #     method = "ETH_CALL"
        #     params {
        #        from: "0xcd2a3d9f938e13cd947ec05abc7fe734df8dd825",
        #        to: "0xcd2a3d9f938e13cd947ec05abc7fe734df8dd826",
        #        data: "0x8300df49"
        #     }
    }
}
```

When the `stableGasPrice` feature is **enabled**, we should adjust the behaviour to retrieve BTC price data according to the he cached _minStableGasPrice_ or the specified method to keep transaction costs stable in fiat currency (like USD or EUR). It first checks if there is a recent, **cached** Bitcoin price within the defined _refresh rate_ and uses this value if available. If no valid cached price exists, it fetches the latest Bitcoin price using the specified method (HTTP_GET or ETH_CALL). 

If the price retrieval fails but a cached price is still valid, the system uses the cached price. If **both** the price fetch and cached price are unavailable, it defaults to the original `minGasPrice` setting.

## Rationale

The mechanism would only trigger if a RSK miner configures the minimumGasPrice in fiat currency by enabling `stableGasPrice` in the configuration settings.

## Backwards Compatibility

This feature would be backwards compatible, as it is just a node behaviour change and it ensures that existing configurations and functionalities remain intact for users who choose not to adopt or enable the `stableGasPrice` setting.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).