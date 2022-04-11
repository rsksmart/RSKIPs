---
rskip: 290
title: Adjust Testnet block minimum difficulty
description: 
status: Draft
purpose: Usa
author: AE (@adrian.eidelman)
layer: Core
complexity: 2
created: 2021-11-18
---
# Adjust Testnet block minimum difficulty

|RSKIP          |290           |
| :------------ |:-------------|
|**Title**      |Adjust Testnet block minimum difficulty|
|**Created**    |18-Nov-2021 |
|**Author**     |AE |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This RSKIP proposes to increase the block minimum difficulty for RSK Testnet. It only impacts Testnet consensus rules, and has no impact in RSK Mainnet.

# **Motivation**

According to RSK Testnet consensus rules, the network's minimum block difficulty will be decreased to a defined value (131,072) if no new blocks are mined for a period of 10 or more minutes. This is a safeguard against unusual increases in hashing power that, in general, drop off the network after some time (for example, this happens when a mining pool is doing tests on RSK Testnet but later switch that hashing power to RSK Mainnet), leaving the network with a block difficulty that can be high for software miners to find new blocks.

The goal of this improvement is to increase the predefined value to 550,000,000. The problem with the current value of 131,072 is that is too low, and even with regular software miners the amount of blocks produced is high until difficulty stabilizes, unnecessarily increasing the workload on the nodes of the network.

# **Specification**

Minimum difficulty for Testnet is defined when Constants class is instantiated as shown in below snippet, extracted from rskj-core/src/main/java/org/ethereum/config/Constants.java file:

```java
public static Constants testnet() {
        return new Constants(
                TESTNET_CHAIN_ID,
                false,
                14,
                new BlockDifficulty(BigInteger.valueOf(131072)),
                new BlockDifficulty(BigInteger.valueOf((long) 14E15)),
                BigInteger.valueOf(50),
                540,
                BridgeTestNetConstants.getInstance()
        );
    }
```

The 4th parameter needs to be set to the new value depending on the block number defined for hard fork activation (not yet defined).


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
