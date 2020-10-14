# External Confirmation Hashrate


|RSKIP          | xxx |
| :------------ |:-------------|
|**Title**      |External Confirmation Hashrate|
|**Created**    |SEP-2020 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |


# **Abstract**

The Strong Fork-aware Merge-mining (SFAMM) protocol proposes a new security model suited for a merge-mined blockchain. If the blockchain implements the SFAMM protocol, and if the assumptions of the model are satisfied, the model assures that large double-spend attacks are prevented, by making attacks visible, attributable, requiring at least 51% Bitcoin miners collusion and, more importantly, requiring a bounded time, where the bound depends on the attacker hashrate but it is known in advance. The bound parameter can be tuned so that in case of attack a community can coordinate actions, such as arranging to manually invalidate a block, or perform a hard-fork, over side channels (such as social networks and forums) in order to deter the attack. 

This RSKIP represents one of the changes to the RSK consensus that is required to achieve security under the SFAMM model.  




## Motivation

The only contribution to the cryptoeconomic security of an independent proof-of-work chain without subsidy is the existence of high transaction fees. In the case of sidechain 1:1 pegged to a primary chain that is protected by merge-mining, there is a high strategic alignment between the primary chain miners and merge-miners, so the honest majority and the fact that attacks to the secondary chain often become attributable are the two main assurances for the security for the sidechain. Nevertheless, RSK went further, and deployed the Armadillo system to provide fork-awareness. This sub0system creates a time window to trigger an emergency response by the community, in case of long reorg attempts.  This addition increased the security of RSK even more. Nevertheless, Armadillo requires a separate communication channel and the existence of at least one honest node monitoring the Bitcoin network. The SFAMM protocol and security model captures the benefits of Armadillo, makes Armadillo decentralized, and formalizes the protocol algorithms, assumptions and guarantees. The SFAMM protocol is ideal for RSK as it focuses not only on the hashrate capability of the attacker but also on the processing time that will be required for a successful attack.

This RSKIP proposes a consensus change that enables RSK nodes to:

- compute the active SHA256 hashrate and detect variations on the active hashrate that are compatible with an ongoing attack

- Accumulate all the active SHA256 hashrate in the RSK blockchain, with independence of the amount of hashrate that is directly involved in merge-mining. This hashrate means more accumulated work, increasing the security against certain double-spend attacks. Also, the hashrate accumulated not only includes 100% of Bitcoin hashrate, but also hashrate from bitcoin forks.

  


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

We extend the list uncles to include, after all RSK block headers , any number of heavyblock elements. To maintain semantic coherence, the *uncles* field is renamed *difficultyContributors*, The field *uncleCount* is renamed as *difficultyContribution*.

The number of elements in the *difficultyContributors* list will be the sum of RSK uncles and heavyblock elements.

To distinguish between an RSK uncle and a heavyblock the first 4 bytes of the RLP-encoded item *F* should be examined. The first byte *F(0)* must be (OFFSET_LONG_LIST+*n*), where n is the number of bytes that encodes the content length. *n* must be 1 or 2. The following *n* bytes are then skipped and the following byte *F(n+1)* is read. This corresponds to the first element of an RLP list. If *F(n+1)* is (OFFSET_SHORT_ITEM + 32), then the element to be read is the parent hash, part of an RSK header, and therefore the encoded item must be processed as an uncle. If *F(n+1)* is (OFFSET_LONG_ITEM + 1) then the element is a Bitcoin block header, part of a mergeMiningProof of a heavyblock, and so the encoded item must be processed as a heavyblock.

The maximum number of uncles referenced in *difficultyContributors* will be reduced from 10 to 6.

The maximum number of heavyblocks referenced in an RSK block will be 2.

Therefore the maximum number of elements contained in *difficultyContributors* will be 8.

### Definitions

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

For the block to be valid, we'll define two auxiliary functions:

**function getInfo(mergeMiningFields)** **returns  (mergeMined,isMMValid)**

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

if (validMerkleTree(merkleRoot,coinbaseTransactionHash)) return (mergeMined,false)
return (mergeMined,false)
```

This logic of the *getInfo()* function is very similar to the existent in the *ProofOfWorkRule* in rskj.

Note that the midstate of *SHA256Digest* class in rskj uses a different format that the one used in this description. We call our variable midstate40 to distinguish between them.

Also it is important to note that the above algorithm requires that for a block that doesn't have the merge-mining the tail size must be equal or grater than 169 bytes, but less than 233 bytes or the midstate is in fact the SHA256 initial state, while for a header that does have the merge mining tag, the tail size must be less than 169 bytes. This last check is the same as how the current check *ProofOfWorkRule* performs as:

```
	remainingByteCount = tail.length - rskTagPosition - 41
	if (remainingByteCount > 128) return (true,false)
```

The reason to require 169 bytes of tail is to prevent an attacker hiding the RSK tag by specifying a starting position after it.

**function isValidHeavyblock(mergeMiningProof) returns (boolean)**

```
if not ProofOfWorkValid(bitcoinMergedMiningHeader) return false
compute btcDifficulty0 from bitcoinMergedMiningHeader
if  (btcDifficulty0 < difficulty) return false
(mergeMined,isMMValid) =getInfo(mergeMiningProof)
if not isMMValid return false
if previouslyIncluded(bitcoinMergedMiningHeader.getBlockHash()) return false
if abs( bitcoinMergedMiningHeader.timeStamp - blockHeader.timeStamp) < 300 return false
if not mergeMined then
    return previouslyIncluded(bitcoinMergedMiningHeader.parent)
else
   return true
```

*ProofOfWorkValid()* checks that the proof of work of the Bitcoin header matches the difficulty specified in the Bitcoin header. The algorithm computes the *btcDifficulty0* specified by Bitcoin header and checks that it is higher than the difficulty of the RSK block that includes it. The call to the function *getInfo()* checks that the *bitcoinMergedMiningMerkleProof* is correct in relation with the other two fields if the mergeMiningProof, according to the same consensus rules used to validate merge-mining.

 The function *previouslyIncluded()* returns true if the bitcoin header with the hash given was included as uncle in any of the previous 256 blocks.

All heavyblocks specified in the uncle vector must be valid.

For a valid heavyblock element we say it is an **anchor heavyblock** if *(mergeMined==true)* . 

If *(mergeMined==false)*, then we say it is a **confirmation heavyblock **. 

### Cumulative Difficulty

Let's call *A* to the difficulty contributed by anchor heavyblocks to the chain cumulative difficulty. 

Let's call *R* to the difficulty contributed by normal RSK blocks whose Bitcoin header difficulty has not reached the *btcDifficulty0*.

Let's call *C* to the difficulty  contributed by confirmation heavyblocks to the chain cumulative difficulty

We propose the following values:

*A=btcDifficulty0/2*

*R=difficulty/2*

*C=btcDifficulty0*

These formulas require using 256-bit arithmetic. This assignment overstates on average the RSK blockchain cumulative difficulty by at most 1/20 (the ratios of average block intervals between RSK and Bitcoin), which we consider too low to use the exact formulas.

The exact values are given by the following formulas:

*q = btcDifficulty0/difficulty*
*R = difficulty\*q/(2\*q-1)*
*A = btcDifficulty0\*q/(2\*q-1)* 

*C=btcDifficulty0*

These formulas may require up to 512-bit arithmetic.

The difficulty contribution is not directly applied to the cumulative difficulty: the values are compressed to the *difficultyContribution* field (which may result in certain loss of precision), and only this amount is contributed back to the cumulative difficulty. This makes the cumulative difficulty easily computed from the *difficultyContribution* and *difficulty* fields.

### Computation of difficultyContribution

To build this field, all contributions in the *difficultyContributors* field are multiplied by 1024, added together, and then divided by the RSK block *difficulty*. Therefore the *difficultyContribution* will be stored scaled by 1024. This guarantees a 0.1% precision for heavyblocks, without taking too much space. For Bitcoin blocks (whose difficulty is approximately 40 times the RSK difficulty), the precision is 0.0025%.  For Bitcoin Cash blocks, with difficulties between 1 and 2 times RSK's, the precision is 0.1%.

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

In Nakamoto consensus, a transaction is confirmed if there are N block confirmations after the transaction has been included. In the SFAMM model, we recommend that economic actors wait also N*v time, where v is the average RSK block interval (currently 30 seconds). 

## Rationale

Adding new fields to the RSK header increases the bandwidth consumption of light nodes. To avoid adding a new field to the RSK block header, the heavyblocks are packed together with uncles in the *difficultyContributors* field. This makes sense semantically as both uncles and heavyblocks have similar roles in the protocol: contribute to the cumulative difficulty of the block.  

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

TBA
# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 
