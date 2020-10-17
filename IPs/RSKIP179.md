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
|**discussions-to**     | |


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

Miners update their timestamp at least once every minute, so 300 seconds (5 minutes) is more than enough for them to provide accurate timestamps. Since generally Bitcoin timestamps are chosen by the miners while the RSK timestamp is chosen by the mining pool server, we only require both timestamps to be close with a high error tolerance.

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

 
