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

This RSKIP proposes a new fork-identification method, **Fork-Aware Consensus (FACON)**, designed to replace the legacy Armadillo security system. FACON provides a fully decentralized, on-chain protocol layer that enables individual nodes to dynamically assess the safety of the blockchain against hidden-fork attacks. By parsing the trail of information left by merge-miners inside Bitcoin `RSK-Tags`, FACON classifies blocks according to their hidden-fork risk profiles, matching the existing safety guarantees, but without relying on centralized off-chain monitoring or alerting infrastructure.

# 1. Motivation

The Rootstock network currently relies on **Armadillo**, an early-warning security system designed to detect hidden forks or severe network partitions. However, Armadillo possesses several structural vulnerabilities that compromise its long-term decentralization and reliability:

- **Centralization of Monitoring:** It requires specialized monitoring software to continuously parse the Bitcoin blockchain. Currently, RootstockLabs is the sole entity known to operate this system.
- **Lack of Incentives:** There are no protocol-level incentives for third parties to run an Armadillo monitoring node, stalling the roadmap toward an automated, decentralized version (Armadillo 2.0).

These limitations can be detrimental to some applications that require extra security assurances, like exchanges, DeFi protocols  or Cross-Chain Bridges. 

FACON replaces Armadillo by encoding fork detection directly into the consensus layer, allowing nodes to inherit Bitcoin's full hashrate security.

# 2. Rationale

An attacker attempting to build a hidden RSK fork must merge-mine their blocks with the BTC network. To do this successfully, they must include a valid `RSK-Tag` inside the coinbase transactions of their successfully mined Bitcoin blocks. This requirement leaves an unalterable audit trail on the Bitcoin blockchain. FACON shifts the network from a reactive, off-chain alerting posture (Armadillo) to a proactive, on-chain verification model by analyzing these tags as merge-mined RSK blocks are included in the blockchain.

### 2.1. Parent Blocks as the Source of Data

While RSK blocks traditionally include the Bitcoin data of their *own* merged-mined block, but this is not enough to determine whether a hidden fork is being mined (since the attacker would never publish their merged-mined blocks on the competing fork). Instead, FACON requires each RSK block to supply data from the parent BTC block of the merged-mined block (`BTCB = BTCA.parent()`) and  a new cryptographic proof demonstrating that that data is correct. 

### 2.2. Uncle Blocks Classification

With the information from the RSK-Tag, it is direct to check whether it references a RSK block that is in the canonical chain or not (this becoming evidence of a hidden fork). The problem with this direct check is that Uncle blocks referenced in the RSK tags would be classified as evidence towards hidden forks. 

The solution for this is to, instead of checking the block (e.g. `RSKX`) referenced inside the RSK-Tag, check its parent block (e.g. `RSKX.parent()`).

This creates another technical issue: Ideally, the protocol would look at the block referenced in the parent Bitcoin block (`RSKX`) and verify if its parent (`RSKX.parent()`) belongs to the canonical RSK chain. However, data inside the Bitcoin `RSK-Tag` is stored in the `hashForMergedMining` format (which strips Bitcoin-specific metadata). Because this hash format does not directly expose the parent block identifier, **nodes cannot look up `RSKX.parent()` directly.**

To solve this, FACON requires nodes to maintain a localized cache of both canonical blocks and valid Uncle blocks. Instead of validating the parent of `RSKX`, the protocol validates **`RSKX` itself** against this combined cache (i.e. `RSKX.parent()`is in the canonical chain if and only if `RSKX` is in the set of canonical chain blocks and Uncle blocks).

# 3. Specifications

This RSKIP constitutes of the following main changes in the protocol:

- A **new proof type**, called Fork-Balance Proof, that shall be included in [RSKIP-351](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP351.md)'s `extendedData` field (Section 3.4. and 3.5.)
- A new `evidenceValue`  component on blocks, included in the Fork-Balance Proof (Section 3.2.).
- A `safetyLevel` measurement of blocks, that will be determined through the values of `evidenceValue`  on validation (Section 3.3.).
- The inclusion of the optional “safe”parameter for existing RPC calls that return blocks, that will use `safetyLevel`  to decide which block to return (Section 3.6.)

## 3.1. Parameters and definitions

| Constant | Default Value | Description |
| --- | --- | --- |
| `EPOCH_LENGTH` | 100 | The sliding window size (in blocks) used to evaluate chain safety. |
| `SAFE_THRESHOLD` | 0.5 | The minimum ratio of supporting blocks required to maintain a SAFE status. |
| `DELAY_PARAMETER` | 60 | A buffer (in seconds) applied to cache eviction calculations. |

For any given Rootstock block `RSKA`:

- `RSKB` = `RSKA.parent()` = The immediate parent block of `RSKA` in the Rootstocks blockchain.
- `BTCA` = `RSKA.btc()` = The Bitcoin block merge-mined with `RSKA`, which may or not be a block on the Bitcoin Blockchain, depending on the difficulty.
- `BTCB` = `BTCA.parent()` = Parent block of `BTCA` , necessarily on the Bitcoin chain when `RSKA` was mined.
- `BTCB.rskReference()` = Rootstocks block merged-mined with `BTCB`, with header referenced in the RSK Tag. In case `BTCB` was not merge-mined with Rootstock, then this is `null` .
- `RSKA.canonicalChain()` = Canonical chain determined by `RSKA` as the tip.

## **3.2. Evidence Classification**

Every validated RSK block `RSKA` is assigned an evidence value (`evidenceValue`) based on the verification of `RSKX = RSKA.btc().parent().rskReference()` against the node's local view of the blockchain:
• **Type 0 (Supportive) [**`evidenceValue = 1`**]:** `RSKX`is found within the node's canonical chain or its verified Uncle cache.
• **Type 1 (Hiding) [**`evidenceValue = -1`**]:** `RSKX` is NOT `null`, but it cannot be found in either the canonical chain or the Uncle cache. This indicates a potential hidden fork.
• **Type 2 (Neutral) [**`evidenceValue = 0`**]:** `RSKX` is `null` (the parent Bitcoin block did not merge-mine RSK).

## **3.3. Safety Level Evaluation**

Every block maintains a running Safety Level (`safetyLevel`) computed over a sliding window defined by `EPOCH_LENGTH`. For a RSK block `RSKA` , it is defined as the sum of the last `EPOCH_LENGTH` values of `evidenceValue` of the chain that has `RSKA` as the tip. 

Let `RSKA.evidenceValue()` be the evidence value of block `RSKA`, and let `facLastEvidenceEpoch` be the evidence value of the last block in `RSKA`'s epoch. The safety level can be calculated as:

`RSKA.safetyLevel() = RSKA.parent().safetyLevel() + RSKA.facEvidenceValue() -RSKA.parent().facLastEvidenceEpoch()`

The block `RSKA` is formally classified as **SAFE** if:

`RSKA.safetyLevel() > SAFE_THRESHOLD * EPOCH_LENGTH.`

For each block `RSKA` we can determine the `lastSafeBlock` , which as the name suggests is the most recent block on the chain determined by `RSKA` that was classified as safe:

```jsx
if RSKA.safetyLevel() > SAFE_THRESHOLD * EPOCH_LENGTH:
	RSKA.lastSafeBlock() = RSKA;
else RSKA.lastSafeBlock() = RSKA.parent().lastSafeBlock();
```

## 3.4. Fork-Balance Proof

The Fork-balance Proof of a RSK block `RSKA`, merged-mined with `BTCA`, in composed of the elements:

- `proofType` (`integer`) = Determines which proof is being provided:
    - `proofType = 0` if `RSKA.evidenceValue = 1` (Supporting).
    - `proofType = 1` if `RSKA.evidenceValue = -1` (Hiding).
    - `proofType = 2` if  `RSKA.evidenceValue = 0` (Neutral).
- `parentBtcHeader` (80 bytes) = Header of  `BTCB = BTCA.parent().header()` .
- `coinbaseHash` (32 bytes) = Double-SHA256 hash of the coinbase transaction of `BTCB` .
- `coinbaseProof` (around 384 bytes) = Merkle proof of inclusion of the coinbase transaction in the Merkle root (`parentBtcHeader.hashMerkleRoot()` ) of transactions in the header.
- `coinbaseLastBytes` (up to 128 bytes) = Tail of the coinbase transaction data, containing the RSK Tag.
- `midstateProof` =   A midstate for the double SHA-256 hashing of the coinbase transaction. Together with the `coinbaseLastBytes` one shall obtain `coinbaseHash`, working as a proof of correctness.

Thus, to allow the verification of the evidence value, each RSK block header extension MUST include a `forkBalanceProof` serialized via RLP:

```jsx
forkBalanceProof = RLP(
    proofType,         // Integer
    parentBtcHeader,   // 80 bytes
    coinbaseHash,      // 32 bytes
    coinbaseProof,     // ~384 bytes
    coinbaseLastBytes, // Up to 128 bytes
    midstateProof      
)
```

## 3.5. Block Header Extension Update

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

## 3.6. JSON-RPC Interface Changes

We shall update the following block retrieval methods:

- `eth_getBlockByNumber()`
- `eth_getBlockByHash()`
- `eth_blockNumber()`

 to support an optional `"safe"` flag parameter:

- **Standard Request:** Returns the requested block at the current canonical tip or height.
- **Safe-Flag Request:** Returns the data payload of the `lastSafeBlock` associated with that block height. If the block is evaluated as safe, it returns itself. If an attack or critical split is underway, it automatically resolves to the last cryptographically secure common ancestor.

### 3.6.1. Parameter Compatibility

In Ethereum/RSK JSON-RPC standard design, the second parameter for `eth_getBlockByNumber` is traditionally a boolean indicating transaction hydration (`hydratedTransactions`). Adding a `"safe"` string as the second parameter breaks compatibility with standard web3 clients.

The best approach on this is to follow **EIP-1898** by allowing the first parameter to be an *object* instead of just a string or number. We could specify the API to accept this:

```jsx
// To get a specific block's safe version:
eth_getBlockByNumber({ "blockNumber": 1500000, "requireSafe": true })

// To just get the current safe tip:
eth_getBlockByNumber("safe")
```

# 4. Backward Compatibility

This modification introduces a breaking change to the block validation and requires a coordinated **hard-fork** activation. All node operators, mining nodes, and client implementations must upgrade their core software prior to the designated activation block height.

# 5. Appendix

## 5.1. Note on Block Data Cache Duration

Since having the Uncle Blocks information is a matter of being able to validate the blocks, I propose keeping a new `facBlocksCache` and `lastBtcBlockTimestamp` , accessible in the `BlockChainImpl` class. This cache would keep all processed blocks that get classified as valid Best Blocks (canonical chain) or valid non-Best Block (Uncles). Blocks in the cache that are too old would be removed from it, but defining the threshold for the removal is a challenge.

A RSK block  `RSKA` to be added to the blockchain will include `BTCA` header, and their timestamp difference will be limited by 300 seconds due to RSKIP-179. Since there is no strict constraint between `BTCA` and `BTCB=BTCA.parent()` timestamps, which means that the best approach is to remove the tail block `RSKX` of the cache at the moment of `RSKA` inclusion when:

`RSKX.timestamp() <  BTC_TAIL(RSKA).timestamp() - 300 - DELAY_PARAMETER` ,

and `BTC_TAIL(RSKA)` is the BTC block with lowest timestamp merged-mined with an RSK block from `RSKA`'s epoch. The rational here is looking for the worst case scenario: When the BTC network lags due to maintenances or other factors, we want the cache to expand since we may need to check for older blocks, so we take the block with oldest timestamp in the ones referenced on `RSKA` epoch, take a limit of 300 seconds for the RSK block  based on RSKIP-179 and finally add a margin for errors and delays with a `DELAY_PARAMETER`  (suggested value of 60 seconds). 

### 5.1.1. Format Adjustment

The RSK block header we read from the RSK Tag comes in the `hashForMergedMining`  format, which means it is the RLP of all fields from a standard RSK header, except for the 3 Bitcoin related fields: `bitcoinMergedMinedHeader` , `bitcoinMergedMiningMerkleProof`  and `bitcoinMergedMiningCoinbaseTransaction` . 

To make our cache of blocks comparable with the header we read inside the RSK Tag, we need to make sure we store block headers that are in the `hashForMergedMining`  format too. Luckily, RSKj already has a built-in method to do this, called `getHashForMergedMining()` . Therefore it is a matter of making sure we use the correct method when implementing the pushing of values to the cache. 

## 5.2. A Note on Reorganizations

When reorganizations happen in the code, after appropriate reversions, the node fill a `toConnect` where it replays the `tryToConnect` function on each block there. This is already enough for the logic behind the Fork-Aware Consensus to work on reorganizations, since a replay will correctly calculate the values of flags and functions. We just need to make sure the parameters used to call `tryToConnect` during a reorganization still trigger the logic of the Fork-Aware Consensus. 

##
