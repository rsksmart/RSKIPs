# Merged Mining Capabilities Extension

|RSKIP          |177           |
| :------------ |:-------------|
|**Title**      |Merged Mining Capabilities Extension |
|**Created**    |APR-2020 |
|**Author**     |MP |
|**Purpose**    |Sca |
|**Layer**      |Node |
|**Complexity** |1 |
|**Status**     |Adopted |

## Abstract

The present RSKIP introduces an extension of merged mining capabilities. It allows summarized data to be hashed as part of the merged mining hash of the block. For this to be possible, network consensus must be changed.

## Motivation

The ability to extend the merged mining process to include information other than the block’s hash in the merged mining hash offers a myriad of new applications to the miner’s hashing power without affecting their performance.

Potential applications may involve new mechanisms for other blockchains to do merged mining with RSK and by transitivity with BTC increasing their level of security.  Moreover, any kind of summarized information can be added to the merged mining hash (if conditions apply) and be validated with PoW security.

## Specification

Recall that the merged mining hash consists of a 20-byte length prefix calculated as the keccak256 hash of the RSK block, followed by a 12-byte length string of information related to Armadillo (introduced in RSKIP110).

The proposed changes are:

1. Introduce a new 20-byte array field in the block called mergeMiningRightHash. This new field will contain the new information that is to be included in the new hash for merged mining. As this new field is now part of the consensus protocol, it will need to be included in the block RLP encoding.
2. Change the way the hash for merged mining prefix is computed. Now instead of only using a hash of the block, a merkle tree will be built using the aforementioned block hash as the left leaf and the new mergeMiningRightHash as the right leaf, and its root will be used as the prefix instead. The following pseudocode illustrates the procedure:

```
def buildHashForMergedMiningPrefix(block):
    oldHashForMergedMiningPrefix = truncate(keccak(block), 20)
    merkleTreeRoot = hash(concatenate(oldHashForMergedMining, mergeMiningRightHash))
    return truncate(merkleTreeRoot, 20)
```

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
