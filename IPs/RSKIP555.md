---

| RSKIP | 555 |
| --- | --- |
| **Title** | Introducing the Fork-Aware Consensus module |
| **Created** | 25-MAY-2026 |
| **Author** | DC (@darcy-camargo), SDL (@SergioDemianLerner) |
| **Purpose** | Sec |
| **Layer** | Core |
| **Complexity** | 2 |
| **Status** | Draft |

# Abstract

This RSKIP proposes a new fork-detection method, **Fork-Aware Consensus (FACON)**, designed to eventually replace the legacy Armadillo security system. FACON provides a fully decentralized, on-chain protocol layer that enables individual nodes to dynamically assess the safety of the blockchain against hidden-fork attacks. By parsing the trail of information left by merge-miners inside Bitcoin RSK tags, FACON classifies blocks according to their hidden-fork risk profiles, matching the existing safety guarantees, but without relying on centralized off-chain monitoring or alerting infrastructure.

# 1. Motivation

The Rootstock network currently relies on **Armadillo**, an early-warning security system designed to detect hidden forks or severe network partitions. However, Armadillo possesses several structural vulnerabilities that compromise its long-term decentralization and reliability:

- **Centralization of Monitoring:** It requires specialized monitoring software to continuously parse the Bitcoin blockchain. Currently, RootstockLabs is the sole entity known to operate this system.
- **Lack of Incentives:** There are no protocol-level incentives for third parties to run an Armadillo monitoring node, stalling the roadmap toward an automated, decentralized version (Armadillo 2.0).

These limitations can be detrimental to some applications that require extra security assurances, like exchanges, DeFi protocols  or Cross-Chain Bridges.

FACON replaces Armadillo by encoding fork detection directly into the consensus layer, allowing nodes to inherit Bitcoin's full hashrate security.

## 1.1 Objective

FACON objective is awareness: to create a usable framework that users can use to increase security over high value or sensitive transactions. Users could then wait to check if any fork is detected before confirming transactions, and/or stop sending sensitive transactions when a detection happen. These actions cap the incentives of the attack.  

FACON is designed against forks that are aiming for reorganization attacks, i.e. long forks.  Short forks, that are mainly product of honest mining with gaps in perception due to network and communication delays, should not be a problem as nodes already wait a minimum time for confirmation accounting for these.

# 2. Rationale

An attacker attempting to build a hidden RSK fork must merge-mine their blocks with the BTC network. To do this successfully, they must include a valid `RSK-Tag` inside the coinbase transactions of their successfully mined Bitcoin blocks. This requirement leaves an unalterable audit trail on the Bitcoin blockchain. FACON shifts the network from a reactive, off-chain alerting posture waiting Armadillo do the work to a proactive, on-chain verification model by  letting nodes themselves analyze these tags.

### 2.1. Parent Blocks as the Source of Data

While RSK blocks traditionally include the Bitcoin data of their *own* merged-mined block, but this is not enough to determine whether a hidden fork is being mined (since the attacker would never publish their own merged-mined blocks on the competing fork). Instead, FACON requires each RSK block to supply data from the parent BTC block of the merged-mined block (`BTCB = BTCA.parent()`), and also to include a new cryptographic proof demonstrating that that data is correct.

### 2.2. Evidence Determination and Uncle Blocks

With the information from the RSK-Tag, it is direct to check whether it references a RSK block that is in the canonical chain or not (this becoming evidence of a hidden fork). The problem with this direct check is that Uncle blocks referenced in the RSK tags would be classified as evidence towards hidden forks.

The solution for this is to, instead of checking the block (e.g. `RSKX`) referenced inside the RSK tag, check its parent block (`RSKX.parent()`).

This creates another technical issue: Ideally, the protocol would look at the block referenced in the parent Bitcoin block (`RSKX`) and verify if its parent (`RSKX.parent()`) belongs to the canonical RSK chain. However, data inside the Bitcoin RSK tag is stored in the `hashForMergedMining` format (which strips Bitcoin-specific metadata). Because this hash format does not directly expose the parent block identifier, nodes cannot look up `RSKX.parent()` directly**.**

To solve this, FACON requires nodes to maintain a localized cache of both canonical blocks and its embedded Uncle blocks. Instead of comparing the parent of `RSKX`, the protocol compares **`RSKX` itself** against this combined cache (i.e. `RSKX.parent()`is in the canonical chain if and only if `RSKX` is in the cache of canonical blocks and Uncle blocks).

# 3. Objective and Applications

First we should understand that the role of FACON is that of bringing awareness, we do not expect nodes to integrate it, at this level, at validation criteria (”should I accept a block?”) or finality criteria (”Should I only count safe blocks for finality?”). 

Blocks that are classified as fork-unsafe are not inherently unsafe themselves. This is a measure of the status of the network for the risk of forks, not of the block. The module is kept as independent as possible, which means that:

- Different nodes can sporadically reach different safety status for the same blocks with no relevant consequences.
- False positives are not problematic if they are not frequent.

Due to this, we expect nodes to use FACON to:

- Use the safety status to determine if they should send or not high value or/and sensitive transactions.
- Use the number of fork-safe/fork-unsafe blocks to determine if they should postpone or not the confirmed status of blocks.
- Use the metrics of detection to establish new thresholds for confirmation.

We do NOT expect nodes to do:

- Filter out blocks classified as unsafe.
- Delay the broadcast and inclusion of proof and challenges.

## 3.1 Extending Confirmation

Here we present a table with the minimum number of RSK blocks that a node need to wait on average for the detection to happen with different levels of confidence. The table consider different levels of strength of the attacker as a percentages of the honest hashrate, and confidence of 90%, 99% and 99.9%. 

| **Attacker Strength   (% honest)** | **Number of blocks  for 90% confidence** | **Number of blocks  for 99% confidence** | **Number of blocks  for 99.9% confidence** |
| --- | --- | --- | --- |
| 101 | 272 | 493 | 716 |
| 110 | 250 | 448 | 632 |
| 125 | 219 | 383 | 546 |
| 150 | 183 | 323 | 468 |
| 200 | 138 | 233 | 330 |
| 300 | 99 | 162 | 224 |
| 500 | 81 | 101 | 135 |

The numbers were obtained by simulating the hidden fork scenario 500.000 times for each 

# 4. Specifications

This RSKIP constitutes of the following main changes in the protocol:

- A **new kind of proof**, called Fork-Balance Proof, that shall be included in RSKIP-351's `extendedData` field (Section 3.4. and 3.5.)
- A new `evidenceValue` value assigned to RSK blocks, derived from the Fork-Balance Proof during validation (Section 3.2.).
- A `safetyLevel` value assigned to RSK blocks that will be determined through the values of `evidenceValue` on validation (Section 3.3.).
- The inclusion of the `requiredSafe` option for existing RPC calls that return blocks. This option will only return blocks with `safetyLevel` above a threshold, or an error when no block is found (Section 3.6.).

## 4.1. Parameters and definitions

| Constant | Default Value | Description |
| --- | --- | --- |
| `EPOCH_LENGTH` | 100 | The number of RSK blocks that are verified to determine the safety level. |
| `SAFE_THRESHOLD` | 0.0 | The threshold parameter that determine if the safety level of the block makes it returnable in `requiredSafe = true` calls. |
| `DELAY_PARAMETER` | 60 | A buffer (in seconds) applied to cache eviction calculations. |

Now we define elements for the better understanding of the specifications. For any given Rootstock block `RSKA`:

- `RSKA.parent()` = The immediate parent block of `RSKA` in the Rootstocks blockchain.
- `BTCA` = `RSKA.btc()` = The Bitcoin block merge-mined with `RSKA`, which may or not be a block on the Bitcoin Blockchain, depending on the difficulty.
- `RSKA.parentCommit() = BTCA.parent()` = Parent block of `BTCA` , necessarily on the Bitcoin chain when `RSKA` was mined. Also called the parent-commit block of `RSKA`.
- `BTCX.rskReference()` = Rootstock block merged-mined with `BTCX`, with header referenced in the RSK Tag. In the case where `BTCX` was not merge-mined with Rootstock, then this is `null` .
- `RSKA.canonicalChain()` = Chain determined by using `RSKA` as the canonical tip.

In case a coinbase transaction contains multiple RSK tags, the one positioned last will be correct one, and thus the one considered in `BTCX.rskReference()`(just like the current validation already behaves).  

!elements (1).png.png)

## **4.2. Evidence Classification**

Every RSK block `RSKA` gets assigned an evidence value (`evidenceValue`) to it during validation. The value is dependent its parent-commit BTC block based on the verification of `RSKX = RSKA.parentCommit().rskReference()` against the node's local view of the blockchain:
• **Type 0 (Supportive) [**`evidenceValue = 1`**]:** `RSKX`is found within the node's canonical chain or its verified Uncle cache.
• **Type 1 (Hiding) [**`evidenceValue = -1`**]:** `RSKX` is NOT `null`, but it cannot be found in either the canonical chain or the Uncle cache. This indicates a potential hidden fork.
• **Type 2 (Neutral) [**`evidenceValue = 0`**]:** `RSKX` is `null` (the parent Bitcoin block did not merge-mine RSK).

Moreover, a proxy evidence value is used when the blocks cache used for the evidence value assignment is not complete, as during node start:

- **Type 3 (Insufficient Cache) [**`evidenceValue = 0`**]:** Cache of RSK blocks and uncles is not complete for comparison.

For such a scenario we assign the evidence value 0 to avoid false-positives and biases.

## **4.3. Safety Level Evaluation**

Every block has a Safety Level (`safetyLevel`) computed over a sliding window of evidence values, with window length defined by `EPOCH_LENGTH`. For a RSK block `RSKA` , it is defined as the sum of the values of `evidenceValue` for the `EPOCH_LENGTH` most recent RSK blocks in `RSKA.canonicalChain()`.  Safety level is only calculated for canonical chain blocks and not for uncle blocks. We deal with the specific situation of reorganizations on Part 5 of this RSKIP. 

Let `RSKA.evidenceValue()` be the evidence value associated to block `RSKA` (calculated using `RSKA.parentCommit()`). If we keep the information of the evidence value for the last RSK block in `RSKA`'s epoch, `facLastEvidenceEpoch`, then the safety level of `RKSA` can be calculated as:

`RSKA.safetyLevel() = RSKA.parent().safetyLevel() + RSKA.facEvidenceValue() -RSKA.parent().facLastEvidenceEpoch()`

The block `RSKA` is formally classified as fork-safe if:

`RSKA.safetyLevel() >= SAFE_THRESHOLD * EPOCH_LENGTH.`

Otherwise, it is classified as fork-unsafe.

### 4.3.1 Available Information

When we determine the evidence value, we use the parent-commit block. The same BTC block is the parent-commit block for multiple RSK blocks due to the difference in hashrates between the chains. 

This means that although the suggested `EPOCH_LENGTH` is 100 (RSK Blocks), the safety level is actually based on around 5 distinct (BTC blocks) counted multiple times. Using BTC to define epoch is not a good solution, because it becomes a problem  to define epochs when orphan blocks or forks happen in BTC. Instead, to make sure users understand the number of considered sources of evidence for the safety value, each display and log of evidence value should be paired with the amount of evaluated BTC blocks.

## 4.4. Fork-Balance Proof

The Fork-balance Proof of a RSK block `RSKA`, merged-mined with `BTCA`, in composed of the elements

- `parentBtcHeader` (80 bytes) = Header of `BTCB = BTCA.parent().header()` .
- `coinbaseHash` (32 bytes) = Double-SHA256 hash of the coinbase transaction of `BTCB` .
- `coinbaseProof` (around 384 bytes, maximum 960 bytes) = Merkle proof of inclusion of the coinbase transaction in the Merkle root (`parentBtcHeader.hashMerkleRoot()` ) of transactions in the header.
- `coinbaseLastBytes` (169 bytes) = Tail bytes of the coinbase transaction data, certainly containing the RSK Tag.
- `midstateProof` (40 bytes)= A midstate for the double SHA-256 hashing of the coinbase transaction. Together with the `coinbaseLastBytes` one shall obtain `coinbaseHash`, working as a proof of correctness.

Thus, to allow the verification of the evidence value, each RSK block header extension MUST include a `forkBalanceProof` serialized via RLP:

```jsx
forkBalanceProof = RLP(
    parentBtcHeader,   // 80 bytes
    coinbaseHash,      // 32 bytes
    coinbaseProof,     // max 960 bytes, expected 384 bytes
    coinbaseLastBytes, // 169 bytes
    midstateProof // 40 bytes
)
```

The logic for midstate proofs is was already discussed in RSKIP-178 and is present in the codebase through the `MinerServerImpl` tools. Coinbase proof limit was set in RSKIP-180.

## 4.5. Block Header Extension Update

This proposal updates the `extendedData` layout introduced in RSKIP-351 and modified in RSKIP-535 to **Version 3 (**`BlockHeaderExtensionV3`). The `forkBalanceProof` is explicitly hashed within the structural root to minimize overhead during processing.

```jsx
extendedData = RLP(version, blockHeaderExtensionHash)

blockHeaderExtensionEncoded = RLP(
    logsBloom,
    txExecutionSublistsEdges,
    baseEvent,
    forkBalanceProof
)

blockHeaderExtensionHash = Keccak256(RLP(
    Keccak256(logsBloom),
    txExecutionSublistsEdges,
    baseEvent,
    Keccak256(forkBalanceProof)
))
```

Headers utilizing V1 or V2 structures after the activation height shall be evaluated as consensus-invalid.

## 4.6 New Validation Filters

| **Filter** | **Rejects if** |
| --- | --- |
| **Header version** | Block header is not **v3.** |
| **Proof present** | Missing or empty `forkBalanceProof` field. |
| **RLP shape** | Proof not decodable as 5 fields: `[parentBtcHeader, coinbaseHash, coinbaseProof, coinbaseLastBytes, midstateProof]` |
| **Parent BTC link** | `mergedMiningHeader.prevHash` ≠ `parentBtcHeader` decoded from `forkBalanceProof` |
| **Coinbase Hash Correctness** | Coinbase hash has empty suffix, incorrect hash length or empty `midstateProof`. |
| Midstate Proof Verification | `midstateProof` and  `coinbaseLastBytes` do not commit to `coinbaseHash`.  |
| Midstate Proof Verification | `coinbaseProof` does not prove `coinbaseHash` `parentBtcHeader` merkle root field. |
|  |  |

## 4.7. JSON-RPC Interface Changes

We shall update the following block retrieval methods:

- `eth_getBlockByNumber()`
- `eth_getBlockByHash()`
- `eth_blockNumber()`

to support an optional `"requireSafe"` flag parameter:

- **Standard Request:** Returns the requested block at the current canonical tip or height.
- **Safe-Flag Request:** Returns the requested block if it is classified as fork-safe, otherwise return an error.

### 4.7.1. Parameter Compatibility

In Ethereum/RSK JSON-RPC standard design, the second parameter for `eth_getBlockByNumber` is traditionally a boolean indicating transaction hydration (`hydratedTransactions`). Adding a `"safe"` string as the second parameter breaks compatibility with standard web3 clients.

The best approach on this is to follow **EIP-1898** by allowing the first parameter to be an *object* instead of just a string or number. We could specify the API to accept this:

```jsx
// To get a specific block's safe version:
eth_getBlockByNumber({ "blockNumber": 1500000, "requireSafe": true })
```

# 5. Attacks and False-Positives Vectors

Fork-aware consensus introduces new modules and elements. To make sure that it only improves the security level of the RSK node, we list the known attacks point out how their damage is mitigated, negated or disincentivized.

## 5.1 Enforcing Unsafe-Blocks as a DoS Attack

Node could, when mining BTC blocks, include fake references to trigger the unsafe status on blocks and delay certain transactions of being included indefinitely. 

Remark: The necessary hash rate for this attack is the same as to perform a reorganization attack, since the enforcing unsafe status means keeping the average of BTC blocks mined my the malicious party above of the honest miners rate. 

In Armadillo **t**he same problem is possible. Armadillo would signal to nodes that there is a hidden fork being mined, delaying partially their interoperability.

Since FACON/Armadillo cap the incentives level to run a reorganization attack, DoS attacks are possible. That said, they are much less harmful than reorganization attacks and extremely costly to keep. The current incentives level does not match the damage of the attack. In either case, FACON or Armadillo are improvements.

## 5.2 Block-Skipping Vulnerability

The RSK blocks only find evidences that are present in the tips of the BTC blockchain, due to how parent-commit blocks are defined. Malicious nodes could try to mine BTC blocks in pairs, to hide the RSK tag with evidence behind a neutral or supportive block. 

This would effectively hide the hidden fork, but at the cost of a huge percentage of adversarial attack power and money, since many of his BTC would be forfeit due to withholding. 

An theoretical estimate of losses according to hashrate percentage of the BTC network can be check below.

| **Attacker Hashrate %** | **Attacker Orphan Rate** | **Relative Revenue Loss** |
| --- | --- | --- |
| **10%** | 97.20% | **-96.90%** |
| **20%** | 89.60% | **-87.20%** |
| **30%** | 78.40% | **-71.03%** |
| **40%** | 64.80% | **-49.88%** |
| **45%** | 57.48% | **-38.49%** |
| **50%** | 50.00% | **-27.28%** |

## 5.4 Orphaned Blocks Create False Positives

If an honest block mine an RSK block that does not win the race to be in the canonical chain and is not included in the winner's list of uncles, it is effectively orphaned. This orphaned block merged-mined BTC block could be included in the BTC canonical chain. In such a situation, each RSK block with that BTC block as the parent-commit will be classified as hiding, even though no hidden fork is happening.

This scenario has low statistical significance, because to happen once many different outcomes need to align, and happening just once does not trigger unsafe-mode. To be relevant, it would need to happen many times in a short time window. But it is good to be aware of it.

# 6. Backward Compatibility

This modification introduces a breaking change to the block validation and requires a coordinated **hard-fork** activation. All node operators, mining nodes, and client implementations must upgrade their core software prior to the designated activation block height.

# 7. Appendix

## 7.1. Note on Block Data Cache Duration

Since having the uncle blocks information is a matter of being able to validate the blocks, we propose keeping a new `facBlocksCache` and `lastBtcBlockTimestamp` , accessible in the `BlockChainImpl` class. This cache would keep all processed blocks that get classified as valid Best Blocks (canonical chain) and also add its emdebbed Uncles list. Blocks in the cache that are too old would be removed from it, but defining the threshold for the removal is a challenge.

A RSK block  `RSKA` to be added to the blockchain will include `BTCA` header, and their timestamp difference will be limited by 300 seconds due to RSKIP-179. Since there is no strict constraint between `BTCA` and `BTCB=BTCA.parent()` timestamps, which means that the best approach is to remove the tail block `RSKX` of the cache at the moment of `RSKA` inclusion when:

`RSKX.timestamp() <  BTC_TAIL(RSKA).timestamp() - 300 - DELAY_PARAMETER` ,

and `BTC_TAIL(RSKA)` is the BTC block with lowest timestamp merged-mined with an RSK block from `RSKA`'s epoch. The rational here is looking for the worst case scenario: When the BTC network lags due to maintenances or other factors, we want the cache to expand since we may need to check for older blocks, so we take the block with oldest timestamp in the ones referenced on `RSKA` epoch, take a limit of 300 seconds for the RSK block  based on RSKIP-179 and finally add a margin for errors and delays with a `DELAY_PARAMETER`  (suggested value of 60 seconds).

### 7.1.1. Format Adjustment

The hash of the RSK block header we read from the RSK tag comes in the `hashForMergedMining` format, which means it is the hash of the RLP of all fields from a standard RSK header, except for the 3 Bitcoin related fields: `bitcoinMergedMinedHeader` , `bitcoinMergedMiningMerkleProof`  and `bitcoinMergedMiningCoinbaseTransaction` .

To make our cache of blocks comparable with this hash from RSK Tag, we need to make sure we store hashes of block headers that are in the `hashForMergedMining` format too. Luckily, RSKj already has a built-in method to do this, called `getHashForMergedMining()`. 

## 7.2. A Note on Reorganizations

Reorganizations happen when a new block classified as IMPORTED_BEST do not have the current best block as its parent. When this happens, the node switch which blocks constitute the canonical chain through the `reBranch()` function. When a `reBranch()` is called, FACON needs to calculate safety levels for the new canonical chain blocks and update the blocks cache. 

This happens exactly because we base the cache on canonical chain blocks plus uncle blocks embedded in the canonical chain blocks. This construction of the uncle lists allow us to minimize divergence between cache content among nodes: agreeing on the main chain means agreeing on the cache, what lead to nodes matching the classification of blocks evidence values. 

## 7.3. A Note on RSKIP-110 Deprecation

This RSKIP may deprecate the fork data elements that were included by RSKIP-110. This would free 12 bytes of the RSK tag, corresponding to:

- 7-byte Commit-to-Parents-Vector (CPV)
- 1-byte number of uncles in the last 32 blocks (NU)
- 4-byte Block Number (BN)

Moreover, this could free all the validation logic for these elements, simplifying the validation logic. 

Removing these elements is NOT part of this RSKIP, as it means that we are deprecating Armadillo, and this proposal can coexist with it for as long as necessary.
