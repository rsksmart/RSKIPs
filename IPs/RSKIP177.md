---
rskip: 177
title: Universal Merged Mining Extension 
description: 
status: Adopted
purpose: Sca
author: SDL (@sergiodemianlerner), MP <mpicco@iovlabs.org>
layer: Node
complexity: 1
created: 2020-04
---
# Universal Merged Mining Extension 

|RSKIP          |177           |
| :------------ |:-------------|
|**Title**      |Universal Merged Mining Extension |
|**Created**    |APR-2020 |
|**Author**     |SDL & MP |
|**Purpose**    |Sca |
|**Layer**      |Node |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

The present RSKIP introduces an extension of merged mining capabilities. The extension modifies the merge mining tag to allow auxiliary data be hashed  together with the RSK block hash. If this auxiliary data is the hash of a Merkle tree of other sidechain block hashes, then this change enables Universal Merge Mining capabilities for RSK.
This RSKIP requires a change to the network consensus rules.

## Motivation

The ability to extend the merged mining process to include information other than the RSK block hash in the merged mining tag (the data following the "RSKBLOCK:" label in Bitcoin coinabse blocks) enables new applications for the use of the miner's hashing power to secure other protocols and blockchains without affecting their mining performance.

Potential applications may involve new mechanisms for other blockchains to do merged mining with RSK and Bitcoin. Moreover, any kind of summarized information can be added to the merged mining hash and be validated with PoW security.

This RSKIP does not specify how other systems will make use of the auxiliary data enabled. That specification should be provided in following RSKIPs. 

## Specification

The merged mining hash consists of a 20-byte length prefix calculated as the keccak256 hash of the RSK block, followed by a 12-byte length string of information related to Armadillo (introduced in [RSKIP110](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP110.md).

The proposed changes are:

1. Introduce a new 20-byte array field in the RSK block header called mergeMiningRightHash. This new field will contain the new information that is to be included in the new merged mining tag. As this new field is now part of the consensus protocol, and it is included in a new field in the block header .
2. Change the way the hash for merged mining prefix is computed. Instead of only using a hash of the block, a merkle tree is built using the RSK block hash  as the left leaf and the new mergeMiningRightHash as the right leaf, and its root keccak256 hash digest is used as in the merge mining tag instead. The following pseudocode illustrates the procedure:

```
def buildHashForMergedMiningPrefix(block):
    oldHashForMergedMiningPrefix = truncate(keccak(block), 20)
    merkleTreeRoot = hash(concatenate(oldHashForMergedMining, mergeMiningRightHash))
    return truncate(merkleTreeRoot, 20)
```

### Changes to the Block header

Because the `mergeMiningRightHash` will almost surely  used as a root for Universal Merge Mining Merkle Tree, the value is included in the RSK block header by the name `ummRoot`.

The RSK block header is currently a list of the following fields:

```
parentHash, unclesHash, coinbase,stateRoot, txTrieRoot, receiptTrieRoot, logsBloom, difficulty, number, gasLimit, 
gasUsed, timestamp, extraData, paidFees, minimumGasPrice, uncleCount [, mergeMiningFields ] 
```

A new ummRoot field is added as the last element before the merge mining fields.

The resulting block header is:
```
parentHash, unclesHash, coinbase,stateRoot, txTrieRoot, receiptTrieRoot, logsBloom, difficulty, number, gasLimit, 
gasUsed, timestamp, extraData, paidFees, minimumGasPrice, uncleCount, ummRoot [, mergeMiningFields ] 
```

If the `ummRoot` field is empty, then the merge mining tag is computed exactly as before, and no additional has is performed. This makes the new system optional until the applications that may use it are identified.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
