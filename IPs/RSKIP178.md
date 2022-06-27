---
rskip: 178
title: External Confirmation Hashrate
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2020-09
---
# External Confirmation Hashrate


|RSKIP          | 178 |
| :------------ |:-------------|
|**Title**      |External Confirmation Hashrate|
|**Created**    |SEP-2020 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     | |


# **Abstract**

The Inclusive Fork-aware Merge-mining (IFAMM) protocol proposes a new security model suited for a merge-mined blockchain. If the blockchain implements the IFAMM protocol, and if the assumptions of the model are satisfied, the model assures that large double-spend attacks are prevented, by making attacks visible, attributable, requiring at least 51% Bitcoin miners collusion and, more importantly, requiring a bounded time, where the bound depends on the attacker hashrate but it is known in advance. The bound parameter can be tuned so that in case of attack a community can coordinate actions, such as arranging to manually invalidate a block, or perform a hard-fork, over side channels (such as social networks and forums) in order to deter the attack. 

This RSKIP represents one of the changes to the RSK consensus that is required to achieve security under the IFAMM model.  


## Motivation

The main contribution to the cryptoeconomic security of an independent proof-of-work chain without subsidy is the existence of high transaction fees. However, in the case of a Bitcoin sidechain that is protected by merge-mining such as RSK, there is a strategic incentive alignment between the Bitcoin miners and merge-miners that results in two additional assurances for the security of the side-chain. First, if the sidechain hashrate is high close to the primary chain hashrate, the honest majority property can be inherited. Second, as Bitcoin miners identify themselves in blocks, attacks to the secondary chain  become attributable either by direct identification of pools that take part of an attack or by the sudden removal of the malicious pools identification information in blocks. Nevertheless, RSK has gone further, and deployed the Armadillo system to provide fork-awareness. Armadillo creates a time window to trigger an emergency response by the community, in case of long reorg attempts. However, Armadillo requires a separate communication channel and the existence of at least one honest node monitoring the Bitcoin network for forks. 
The IFAMM protocol (which this RSKIP is only a part of) rests on a security model that captures the benefits of Armadillo, but decentralizes it. The IFAMM protocol is ideal for RSK as it focuses not only on the hashrate capability of the attacker but also on the processing time that will be required for a successful attack.

This RSKIP proposes a consensus change that enables RSK nodes to:

- compute the active SHA256 hashrate and detect variations on the active hashrate that are compatible with an ongoing attack

- Accumulate all the active SHA256 hashrate in the RSK blockchain, with independence of the amount of hashrate that is directly involved in merge-mining. This additional hashrate that confirms sidechain blocks increases the security double-spend attacks based on long blockchain reorganizations. Also, the hashrate accumulated may not only capture 100% of Bitcoin hashrate, but also capture hashrate from bitcoin forks.

  


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

We define a **mergeMiningProof** to be a list containing the *mergeMiningFields*.

The block in encoded as:

```
blockHeader, transactions, uncles
```

And *uncles* is a list of RSK block headers. We define a **heavyblock** to be a *mergeMiningProof* with certain properties that will be verified in consensus.

We extend the list of uncles to include, after all RSK block headers, a number of heavyblock elements (defined later). To maintain semantic coherence, the *uncles* field is renamed *difficultyContributors*. The field *uncleCount* is renamed as *difficultyContribution*.

*difficultyContributors* will be an RLP list two two elements: the first is the old *uncles* list. The next is the new *heavyblocks* list.

```
difficultyContributors  = { uncles  , heavyblocks }
```

*difficultyContributors* will also be an RLP list of two elements: the first, called *immediateContribution*, represents the difficulty contributed by uncles. The second, called *delayedContribution*, represents the contribution made by confirmation heavyblocks.

```
difficultyContributors = { immediateContribution , delayedContribution }
```

The maximum number of uncles referenced in *uncles* will be reduced from 10 to 6.

The maximum number of heavyblocks referenced in *heavyblocks* will be 2.

A Bitcoin block header contains these fields:

| Field          | Purpose                                                      | Size (Bytes) |
| -------------- | ------------------------------------------------------------ | ------------ |
| Version        | Block version number                                         | 4            |
| hashPrevBlock  | 256-bit hash of the previous block header                    | 32           |
| hashMerkleRoot | 256-bit hash based on all of the transactions in the block   | 32           |
| Time           | Current [block timestamp](https://en.bitcoin.it/wiki/Block_timestamp) as seconds since 1970-01-01T00:00 UTC | 4            |
| Bits           | Current [target](https://en.bitcoin.it/wiki/Target) in compact format | 4            |
| Nonce          | 32-bit number                                                | 4            |

We define how to convert the Bits field into a target difficulty that can be compared with the RSK block difficulty.

The Bits field is stored in a compact format. Let Bits be a byte array *{ E,Q1,Q2,Q3 }*. Let M be the integer value *(Q1 << 16 + Q2 << 8 + Q3)* (that is the *Q* array interpreted in big-endian). The Bitcoin target is computed as *M\*2^(8(E-3))*. We define the *btcDifficulty0 = 2^256 / target*. Note that the *btcDifficulty0* is *2^32* times higher than the Bitcoin block difficulty.

### Validation

We define the function *getPoWInfo()* to check the validity of all kinds of blocks (mainchain, uncles and elements in the *heavyblocks* list). This means that the function that checks the validity of the proof of work of normal RSK blocks is changed to call getPoWInfo(). This is because some of the RSK blocks can be anchor heavyblocks.

First we define an auxiliary function *getMMInfo()*:

**function getMMInfo()** 

Arguments:

* **mergeMiningFields**

Returns:

* boolean **mergeMined**
* boolean **isMMValid**

```
x = bitcoinMergedMiningCoinbaseTransaction of mergeMiningFields

midstate40 = fist 40 bytes of x
tail = the remaining bytes of x skipping the first 40 bytes

lastTag = last Index Of "RSKBLOCK:" in tail
byteCount = bigEndianToInt64(extract 4 bytes of midstate40 starting at offset 8)
mergeMined = (lastTag>=0)

if (not mergeMined) then
    if ((tail.length>=169) and (tail.length<233)) or (byteCount==0)
		return (false,false)
	else
		return (false,true)
else	
	expected = "RSKBLOCK:"+ RSKHashForMergedMining
	rskTagPosition = last Index Of expected in tail
	if (rskTagPosition == -1)  return (true,false)
	if (rskTagPosition >= 64)  return (true,false)
	if (rskTagPosition !=lastTag) return (true,false)
	if (tail.length - rskTagPosition > 169) return (true,false)

coinbaseLength = tail.length + byteCount;
if (coinbaseLength <= 64) return (mergeMined,false)

transactionHash = SHA256(tail) starting from the midstate40 
coinbaseTransactionHash = reverseBytes(transactionHash)

if (not validMerkleTree(merkleRoot,coinbaseTransactionHash)) return (mergeMined,false)
return (mergeMined,true)
```

This logic of the *getMMInfo()* function is very similar to the existent in the *ProofOfWorkRule* in rskj and the *ProofOfWorkRule* must be changed to use this function.

If (isMMValid==true) and (mergeMined==true), then the input represents an **anchor heavyblock** (it can be an also an internal uncle, and external uncle, or a mainchain block). 

If (isMMValid==true) and (mergeMined==false), then the input represents an **confirmation heavyblock**. 

Anchor heavyblocks can be referenced by three distinct means:

- in normal mainchain RSK block, using the mainchain block merge-mining proof
- in an RSK uncle block header, using the uncle's merge-mining proof, representing a **internal uncle anchor heavyblock**. 
- in the *heavyblocks* list, also representing an **external uncle anchor heavyblock**, but one that has been  extracted from the Bitcoin blockchain, and not from the RSK network.

To identify **external uncle anchor heavyblocks** from the Bitcoin blockchain, the ARMADILLO data must be extended to include at least 14 bytes of the RSK parent block id, or 4 bytes of the Bitcoin block id associated with it. Until ARMADILLO is improved, this RSKIP proposes that the ARMADILLO tag as it is today is used to identify the uncle. This will extend the uncle reach to a depth of 64 at most, which facilitates "nothing-at-stake" double-spends for up to depth 64. By including just two bytes of the Bitcoin pow of the prior RSK block in the ARMADILLO data, the attack probability is reduced 2^16-fold. In other words, considering 50% of the hashrate merge-mining RSK, the attack opportunity will only occur spontaneusly once every 2.5 years.

Note that the midstate of *SHA256Digest* class in rskj uses a different format that the one used in this description. We call our variable midstate40 to distinguish between them.

Also it is important to note that the above algorithm requires that for a block that doesn't have the merge-mining the tail size must be equal or greater than 169 bytes, but less than 233 bytes or the midstate is in fact the SHA256 initial state, while for a header that does have the merge mining tag, the tail size must be less than 169 bytes. This last check is the same as how the current check *ProofOfWorkRule* performs as:

```
	remainingByteCount = tail.length - rskTagPosition - 41
	if (remainingByteCount > 128) return (true,false)
```

The reason to require 169 bytes of tail is to prevent an attacker hiding the RSK tag by specifying a starting position after it.

**function getPoWInfo()**

Arguments:

* **rskBlockHeader,**
* **mergeMiningProof,**
* boolean **fromHeavyblocksArray,**
* boolean **mainchain,**
* boolean **uncle**

Returns:

- boolean **isValid**
- uint **immediateDifficultyContribution**
- uint **delayedDifficultyContribution**
- boolean **includeInBtcBlockReferences**

```
isValid = false
immediateDifficultyContribution =0
delayedDifficultyControbution =0
includeInBtcBlockReferences = false

if not fromHeavyblocksArray
	// all RSK blocks (uncles or mainchain) must have valid proof-of-work at the
	// RSK block difficulty
	if not rskProofOfWorkValid(rskBlockHeader,bitcoinMergedMiningHeader) then return

(mergeMined,isMMValid) =getMMInfo(mergeMiningProof)
if not isMMValid return

isHBTimeValid = abs( bitcoinMergedMiningHeader.timeStamp - rskBlockHeader.timeStamp) < 300
if not isHBTimeValid return

btcDifficulty0 =getBtcDifficulty0(bitcoinMergedMiningHeader)
isPotentialHeavyBlock=(btcDifficulty0 >= difficulty) 
if  (isPotentialHeavyBlock) 
	isHeavyblock = btcProofOfWorkValid(bitcoinMergedMiningHeader) return
else
	isHeavyblock = false

if not mergemined
	// This can only be a confirmation heavyBlock
	if (not fromHeavyblocksArray) return
	if (not isHeavyblock) return
	// We check that there is a btc parent registered
	if btcBlockpreviouslyIncluded(bitcoinMergedMiningHeader.getBlockHash()) return
	delayedDifficultyControbution = C(rskBlockHeader,bitcoinMergedMiningHeader)
	includeInBtcBlockReferences = true
	
if not isHeavyblock and mergemined
	if (fromHeavyblocksArray) return
	// it is a normal block or uncle block
	immediateDifficultyContribution = R(rskBlockHeader,bitcoinMergedMiningHeader)
    if not previouslyIncluded(bitcoinMergedMiningHeader.parent) return
    // uncles are already checked for previous inclusion, and uncle inclusion depth
    // is shorter than IFAMM depth, so we don't need to check again.
    // This check should not change anything, and it can be used as an assertion
    if btcBlockpreviouslyIncluded(bitcoinMergedMiningHeader.getBlockHash()) return
	
if isHeavyblock and mergemined
	// this is an:
	// * internal uncle anchor heavyblock or
	// * external uncle anchor heavyblock
    // * mainchain anchor heavyblock
    includeInBtcBlockReferences = true
	if (fromHeavyblocksArray)
		// external uncle anchor heavyblock
		immediateDifficultyContribution = C(RSKblockHeader,bitcoinMergedMiningHeader)
	else
		// internal anchor heavyblock or uncle anchor heavyblock
		immediateDifficultyContribution = A(rskBlockHeader,bitcoinMergedMiningHeader)
else
	// elements in fromHeavyblocksArray must be blocks solved at Bitcoin difficulty 
	if fromHeavyblocksArray return
	
isValid = true
return
```

In the shown pseudo-code *btcProofOfWorkValid()* checks that the proof of work of the Bitcoin header matches the difficulty specified in the Bitcoin header. The algorithm computes the *btcDifficulty0* specified by Bitcoin header and checks that it is higher than the difficulty of the RSK block that includes it. The call to the function *getMMInfo()* checks that the *bitcoinMergedMiningMerkleProof* is correct in relation with the other two fields if the mergeMiningProof, according to the same consensus rules used to validate merge-mining.

Every merge-mining proof,  in mainchain, in the uncles list or in the heavyblocks list must be validated with the *getPoWInfo()* method. If the method returns isValid==false, then the block is invalid. If the method returns *iincludeInBtcBlockReferences*==true, then this means it is an anchor or confirmation heavyblock and therefore the associated Bitcoin block hash is must be included to the list *btcBlockReferences* that can be referenced by *btcBlockPreviouslyIncluded()*. 

The function *btcBlockPreviouslyIncluded()* returns true if the bitcoin header with the hash given was included in the  *btcBlockReferences*  list. This array maintains a list of all included heavyblocks in the past 256 blocks. The array need not to be stored, as it can be recomputed each time from the heavyblocks, uncles and mainchain blocks in the past 256 blocks. Note that references may form a DAG, and therefore an included heavyblock may have more than one child in the last 256 blocks. 

### Cumulative Difficulty

Let's call *A* to the difficulty contributed by anchor heavyblocks to the chain cumulative difficulty. 

Let's call *R* to the difficulty contributed by normal RSK blocks whose Bitcoin header difficulty has not reached the *btcDifficulty0*.

Let's call *C* to the difficulty  contributed by confirmation heavyblocks to the chain cumulative difficulty.

A, R, C are functions of the current RSK block header and the Bitcoin block header being referenced by the merge-mining fields.

We propose the following functions:

*A=btcDifficulty0/2*

*R=difficulty/2*

*C=btcDifficulty0*

These formulas require the use of 256-bit arithmetic. This assignment overstates on average the RSK blockchain cumulative difficulty by at most 1/20 (the ratios of average block intervals between RSK and Bitcoin), which we consider thie error low enough to prefer the simplified formulas.

The exact values are given by the following formulas:

*q = btcDifficulty0/difficulty*
*R = difficulty\*q/(2\*q-1)*
*A = btcDifficulty0\*q/(2\*q-1)* 

*C=btcDifficulty0*

These formulas may require up to 512-bit arithmetic.

Note that R and C contribute to de *immediateContribution* field, but A contributes to the *delayedContribution* field. The difficulty contributions is not directly applied to the cumulative difficulty: the values are compressed to the *difficultyContribution* field (which may result in certain loss of precision), and only this amount is contributed back to the cumulative difficulty. This makes the cumulative difficulty easily computed from the *difficultyContribution* and *difficulty* fields.

The *delayedContribution* is applied to the cumulative difficulty 40 blocks later. 

### Computation of difficultyContribution

To build the *immediateContribution* and *delayedContribution* fields, their real values are multiplied by 1024, added together, and then divided by the RSK block *difficulty*. Therefore the *difficultyContribution* fields will be stored scaled by 1024. This guarantees a 0.1% precision for heavyblocks, without taking too much space. For Bitcoin blocks (whose difficulty is approximately 40 times the RSK difficulty), the precision is 0.0025%.  For Bitcoin Cash blocks, with difficulties between 1 and 2 times RSK's, the precision is 0.1%.

### Sha256Digest Class Midstate

The *Sha256Digest* midstate is 52 bytes in length. To initialize to work for RSK merge-mining it requires that the first 8 bytes and the last 4 bytes are set to zero. This is because *Sha256Digest* midstate enables starting from any point in the stream, while the RSK protocol requires the starting point matches a 64-byte boundary. 

```
Sha256Digest mapping of midstate:
0: xBuf (4-bytes array) zeroed by RSK (buffer used to process sub-word data)
4: xBufOff (int32) zeroed by RSK (offset of the xBuf that is filled)
8: byteCount (int64) <--- midstate40 starts here
16: H1 (uint64)
20: H2 (uint64)
24: H3 (uint64)
28: H4 (uint64)
32: H5 (uint64)
36: H6 (uint64)
40: H7 (uint64)
44: H8 (uint64) <--- midstate40 ends after this field
48: xOff (int32) zeroed by RSK (number of words (mod 64) following the misdtsate that have not been updated into the hash)
```

### Transaction Confirmation

In Nakamoto consensus, a transaction is confirmed if there are *N* block confirmations after the transaction has been included. In the IFAMM model, we recommend that economic actors wait also *N\*v* time, where *v* is the average RSK block interval (currently 30 seconds). 

## Rationale

Several design choices have been taken. This section describes the most relevant.

#### Changing the Block Header

Adding new fields to the RSK header increases the bandwidth consumption of light nodes. To avoid adding a new field to the RSK block header, the heavyblocks are packed together with uncles in the *difficultyContributors* field. This makes sense semantically as both uncles and heavyblocks have similar roles in the protocol: contribute to the cumulative difficulty of the block.  

#### Limiting the number of heavyblocks

There are two reasons to have more than one heavyblock referenced in the *difficultyContributors* field:

1.  To accept heavyblocks from Bitcoin forks, in that case that more than one Bitcoin-like blocks are created in the same RSK block interval. However, this case would be rare. Most Bitcoin-forks have the same block average interval as Bitcoin, therefore the risk of many heavyblocks being queued is extremely low. 
2. To accept transferring proof of work from Bitcoin to RSK in case RSK merge-mining is paused due to a network health problem. In that case, it's possible that confirmation heavyblocks accumulate during a certain period, and therefore they are enqueued to be added to RSK blocks, and RSK blocks needs to be able to process them as fast as possible. However, accepting old confirmation heavyblocks can also be a vector of attack, for example, if there is a certain Bitcoin fork that RSK miners are not aware of, and this fork is producing blocks, then selfish RSK fork may try to reference hundreds of past confirmation heavyblocks from this hypothetical fork, and reorganize the RSK blockchain. Therefore this RSKIP requires RSK block timestamps to be close to confirmation heavyblock timestamps also. 

While an average RSK block now transfers more difficulty to the blockchain than a Bitcoin Cash block, this is due to the existence of uncle blocks. Although the current RSK hashrate is 23 times higher than Bitcoin Cash, the actual difficulty of an RSK block is slightly lower than a Bitcoin Cash block. Therefore RSK could reference Bitcoin Cash heavyblocks. 

#### Delaying the difficulty contribution of confirmation heavyblocks

The *delayedContribution* is applied to the cumulative difficulty after 60 blocks. This is to deter an attack and an accidental situation. If all difficulty contributions were immediate, then:

* When a malicious miner sees a confirmation heavyblock in Bitcoin, it can create a block template having a parent block which is 10 blocks behind the best block, and, if he finds the next block first, it  immediately reverts 10 blocks of the blockchain, because the heavyblock contributes with a difficulty that is higher than 10 normal RSK blocks.
* If the Bitcoin network creates a stale block, some of the miners may see it while some others may not. If  a malicious miner sees the Bitcoin stale that is a confirmation heavyblock block but nobody else does, he has an advantage to to revert the RSK blockchain, and perform a double-spend attack by referencing that heavyblock in his private chain. 

By delaying the contribution the first problem is eliminated, because the honest fork will also reference the confirmation heavyblock before the malicious fork increases its difficulty.

The second problem is mitigated because with the delay the attacker needs to catch up with the honest chain by mining selfish blocks to realize the advantage. The advantage may be about 40 blocks (because of the contribution made by the stale block) but the attacker also gets a 1 to 40 block disadvantage because he must revert a confirmed transaction in order to double-spend. The higher the payment amount, the more blocks the victim will be waiting for confirmation. Simulations show that with a 60 block delay in the difficulty transfer, the attacker would still require over 40% of RSK hashrate to attempt a double-spend that reverts 20 blocks.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers should also be updated.

## Test Cases

TBD

## Implementation

The rskj code would need to modified in at least the following places:

- co/rsk/core/DifficultyCalculator.java* must be modified. *getBlockDifficulty()* now receives the *difficultyContribution* field.
- *co/rsk/core/bc/SelectionRule.java* must be modified. *isBrokenSelectionRule()* currently considers uncle counts, and it should consider the *difficultyContribution*. 
- *co/rsk/remasc/Sibling.java* must be modified in the same way as *SelectionRule.java*.
- co/rsk/mine/ForkDetectionDataCalculator.java must be modified. Armadillo 2 does not need this value. Therefore if Armadillo 2 is implemented prior this RSKIP, the number of uncles will no longer be present or validated for Armadillo, as Armadillo 2 will contain the full cumulative difficulty packed into the same space. If this RSKIP is implemented before Armadillo 2, then the uncle count can still be validated, although it will not provide any benefit for Armadillo fork detection module.

## Security Considerations

This RSKIP protects the RSK network from a large gamut of double-spend attacks by either increasing the their cost (reaching Bitcoin levels of cryptoeconomic security) or by forcing the attacker to be public about the attack during the whole payment confirmation interval, leaving a trace of anchor heavyblocks in Bitcoin blocks and reducing the amount of cumulative hashrate transferred to RSK.

However, some variants of the double-spend attack, such as isolating a victim node,  still need additional changes to be prevented. These other changes, required by the IFAMM model, will be specified in other RSKIPs.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 
