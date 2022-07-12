---
rskip: 179
title: BTC-RSK timestamp linking
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2020-10
---
# BTC-RSK timestamp linking


|RSKIP          | 179 |
| :------------ |:-------------|
|**Title**      |BTC-RSK timestamp linking|
|**Created**    |OCT-2020 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |
|**discussions-to**     |https://research.rsk.dev/t/rskip-179-btc-rsk-timestamp-linking/44 |


# **Abstract**

The Strong Fork-aware Merge-mining (SFAMM) protocol proposes a new security model suited for a merge-mined blockchain. If the blockchain implements the SFAMM protocol, and if the assumptions of the model are satisfied, the model assures that large double-spend attacks are prevented, by making attacks visible, attributable, requiring at least 51% Bitcoin miners collusion and, more importantly, requiring a bounded time, where the bound depends on the attacker hashrate but it is known in advance.  

This RSKIP represents one small change to the RSK consensus that is required to achieve security under the SFAMM model.  


## Motivation

This RSKIP aims to prevent a merge-miner attacker from mining old RSK blocks with the current Bitcoin hashrate. The RSKIP proposes to prevent the Bitcoin block timestamp to be far away from the RSK timestamp. Since the Bitcoin timestamp is partially malleable for a minority miner, generally up to 1 hour in the past, the prevention only works for reversion attacks that span of large time scales of at least several hours.  


## Specification

 The RSK block header is a list of the following elements:

```
parentHash, unclesHash, coinbase,stateRoot, txTrieRoot, receiptTrieRoot, logsBloom, difficulty, number, gasLimit, gasUsed, timestamp, extraData, paidFees, minimumGasPrice, uncleCount, ummRoot [, mergeMiningFields ] 
```

The *mergeMiningFields* are the following:

```
bitcoinMergedMiningHeader,
bitcoinMergedMiningMerkleProof,
bitcoinMergedMiningCoinbaseTransaction
```

A Bitcoin block header contains these fields:

| Field          | Purpose                                                      | Size (Bytes) |
| -------------- | ------------------------------------------------------------ | ------------ |
| version        | Block version number                                         | 4            |
| hashPrevBlock  | 256-bit hash of the previous block header                    | 32           |
| hashMerkleRoot | 256-bit hash based on all of the transactions in the block   | 32           |
| timestamp      | Current [block timestamp](https://en.bitcoin.it/wiki/Block_timestamp) as seconds since 1970-01-01T00:00 UTC | 4            |
| bits           | Current [target](https://en.bitcoin.it/wiki/Target) in compact format | 4            |
| nonce          | 32-bit number                                                | 4            |

### Validation

Every block in the mainchain, and every uncle block referenced must verify the following rule: 

```
abs( bitcoinMergedMiningHeader.timestamp - rskBlockHeader.timestamp) < 300
```

*Abs()* function returns the absolute value. Values are expressed in seconds.

## Rationale

We assume that economic nodes wait a number of blocks for confirmation when receiving a payment. The number of blocks can be fixed or depend on the amount of the payment, but it’s unrelated to the cumulative difficulty. There are two variants of the double-spend attack depending on the attack timing: an attack that starts after the victim confirms the transaction or an attack that starts before. The first is called Reversion-After-Confirmation (RAC), and the second Reversion-During-Confirmation (RDC). The distinction is of uttermost importance. RSK already has tools to detect and avoid RDC attacks. The Armadillo systems monitors the Bitcoin blockchain and alerts of parallel forks. Also nodes can monitor the total active hashrate and detect sudden reductions. The SFAMM consensus changes (not discussed in this RSKIP) provide also a new method to prevent RDC attacks. However, in a RAC attack, the attack starts after the attacker has already cashed out the first spend, and only the second (double) spend remains. To revert the blockchain after the cash out, the attacker needs to mine blocks with timestamps in the past or leave large time gaps between blocks. It's easy to penalize forks containing timestamp gaps, as SFAMM proposes, so that the cumulative work required by the attacker is much higher. The only option left to the attacker is to mine with past timestamps.

RSK has access to a cryptoeconomic timestamping system, the timestamps of the Bitcoin blocks need to be approximately accurate for the blocks to by accepted by Bitcoin consensus, but RSK is not currently using this feature. In this RSKIP we propose forcing the RSK timestamp to match the Bitcoin timestamp with an error margin of 5 minutes. If the merge-miner wants to cheat, it must give up the profit from Bitcoin mining and create invalid Bitcoin blocks.

Miners normally update their timestamp at least once every minute, so 5 minutes is a loose tolerance. Since generally Bitcoin timestamps are chosen by the miners while the RSK timestamp is chosen by the mining pool server, RSK shouldn't impose a lower tolerance.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers do not need to be updated. 

## Test Cases

1. bitcoinMergedMiningHeader.timestamp = 1300 and rskBlockHeader.timestamp = 1000
   Fails validation

2. bitcoinMergedMiningHeader.timestamp = 1300 and rskBlockHeader.timestamp = 1001
   Passes validation
3. bitcoinMergedMiningHeader.timestamp = 1001 and rskBlockHeader.timestamp = 1300
   Passes validation

## Implementation

The rskj code would need to modified to add this rule. This rule can be added as a separate class or modifying the *BlockTimeStampValidationRule* class. 

## Security Considerations

This RSKIP protects the RSK network from reduced set of double-spend attacks forcing the attacker to mine a private fork before the victim of a transaction has been confirmed for the first time, and only if the victim waits a number of confirmation blocks higher 120. 

Other variants of the double-spend attack still need additional changes to be prevented. These other changes, required by the SFAMM model, will be specified in other RSKIPs.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 
