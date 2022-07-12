---
rskip: 223
title: Cumulative Work in Fork Detection Data
description: 
status: Draft
purpose: Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-03-31
---
# Cumulative Work in Fork Detection Data

|RSKIP          |223           |
| :------------ |:-------------|
|**Title**      |Cumulative Work in Fork Detection Data|
|**Created**    |31-MAR-2021 |
|**Author**     |SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     |https://research.rsk.dev/t/rskip-223-cumulative-work-in-fork-detection-data/150|

# **Abstract**

This RSKIP proposes an improvement to the Fork Detection Data present in merge-mining tags (used by the Armadillo Fork Detection System). The tag is extended to include the cumulative work of the chain. 

# **Motivation**

The RSK merge-mining tags cointains the following fields:

* 7-byte Commit-to-Parents-Vector (CPV)
* 1-byte number of uncles in the last 32 blocks (NU)
* 4-byte Block Number (BN)

The last 3 fields are the Fork Detection Data, as specified in [RSKIP110](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP110.md).

The field NU (number of uncles) is used by Armadillo to detect forks containing cumulative work masqueraded as uncle blocks. 
However, is was shown that setting alerts based on this field is difficult because cumulative work has to be guessed.
We propose to extend the merge-mining tag to from 32 bytes to 40 bytes to include a field specifying the cumulative work.
This change not only improves the security against partially hidden forks, but more importantly it provides information about unintended forks. It enables the measurement of the risk that an unintended fork surpasses the honest chain by making the fork cumulative difficulty visible.

This change may impact mining pool softwares which expect a fixed size block hash field from mnr_getWork(). To reduce the need to patch those softwares in a short time, the proposal specifies that after activation both the old and new merge-mining tags can coexists, and we expect that the old format to be phased out in a following hard fork.


# **Specification**

The new merge-mining tag format is the following:

* PREFIX: 20-byte prefix of the hash-for-merge-mining
* CPV: 7-byte Commit-to-Parents-Vector (CPV)
* NU: 1-byte number of uncles in the last 32 blocks (NU)
* VER_BN4: packed field
* BN:  3-byte Block Number
* CD: 8=byte commulative difficulty (only available in the new tag format)


Version 0 represents the old tag format. To enable the new tag-format, version must be set to 1.

the field VER_BN4 contains two nibbles. The hi nibble contains the tag version. Thw low nibble containst the highest 4 bits of the BN field.
CD is specified in units of 2^60 operations. 

If the BN field or the CD field overflows, it wraps around zero.


# Rationale

The granularity of the CD field permits specifying a cumulative difficulty up to 2^124 operations. Currently RSK has a cumulative difficulty of about 2^92 operations, so the cumulative difficulty can still grow 2^32 times without overflow.
The minimum difficulty that can be tracked is 2^60. The current difficulty of an RSK block is about 2^70 operations, which means that the tag format can still be usefull even if the difficulty of the RSK blockchains is reduced 1000 times. 
The BN field space is restricted from 32 to 28 bits, so it will wrap around zero at block 268,435,456. Currently the block number is 3,230,186, which means that the wrap will occur in approximately 254 years. Fork monitoring applications can easily deduce if the field has wrapped and how many times it has based on the Bitcoin block timestamp.


An alternative to this proposal is [RSKIP224](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP224.md). RSKIP224 proposes the re-interpretation the CPV field so that the jumps are considered not over the block numbers, but over the virtual array of blocks and uncles previously referenced. This means that a fork containing many uncles will sooner show a divergence point in the CPV, and therefore cannot be hidden. RSKIP224 provides some of the same benefits of this proposal, but is ortoghonal to it. Both can be combined. 


Switching to the new tag format could incentivized by penalizing miners using the old tag format after the activation of this RSKIP. For example, they could  receive only 50% ofthe rewards paid by the REMASC contract. This should be specified in another RSKIP.

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers that are always in sync do not need to be updated. 

# Test Cases

TBD

## Security Considerations

This cumulative difficulty present in the new tag should not be used alone to trigger an alert. A malicious minority miner can lie and create an RSK merge-mining tag with false data pretending the existence of a hidden fork of arbitrary high difficulty. The miner considered in this attack is not participating in RSK merge-mining (as his blocks are invalid) but only tries to disrupt the RSK network. Therefore nodes must use the cumulative difficulty information together with the detection of other merge-mining blocks (linked to the fork using the CPV field) to estimate the real hashrate pariticpating in the malicious fork.
The new field does allow nodes to dismiss an unintended fork created by single RSK miner in isolation. If the miner has substancial but not the majority of the hasrate, the fork may look risky because of its length, but it is not so because it doesn't collect the difficulty of uncles produced by the rest of the network.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
