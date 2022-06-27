---
rskip: 89
title: Add Bitcoin block query methods to the bridge contract
description: 
status: Adopted
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2018-07-
---
# Add Bitcoin block query methods to the bridge contract

|RSKIP          |89           |
| :------------ |:-------------|
|**Title**      |Add Bitcoin block query methods to the bridge contract|
|**Created**    |JULY-2018 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

# **Abstract**

The bridge does not provide methods for the location and retrieval of information related to a specific block in the bridge storage.
Extracting information from the Bridge view of the Bitcoin blockchain ise useful for numerous applications, such as p2p swaps.
The current RSKIP proposes that the new methods are limited to be called offchain.
However, this same methods may be in opened to smart-contracts in the future, creating a reliable Bitcoin blockchain oracle.

# **Motivation**

Create the foundations for making available a bitcoin blockchain oracle to RSK smart-contracts.

# **Specification**

Three new methods are added to the RSK bridge:

* int getBtcBlockchainInitialBlockHeight()

Returns the first block that is held by the Bridge contract. The bridge contract does not hold all Bitcoin blocks starting from genesis, but a tail. The first block to include was decided when RSK genesis block was created.
The cost of this method call is 20K gas.


* string[] getBtcBlockchainBlockLocator()

Returns a list of at most 102 block hashes starting from the last, and going backwards with exponentially increasing distances.
The first block hash in the returned array is the last block. The remaining block hashes correspond to the block heights computed as (bestBlockHeight - 2^i), for i starting at 0.
If the height lies below the initial block height, then it is not returned.
The last elements is always the initial block hash.
The cost of this method call is 76K gas.

* bytes getBtcBlockchainBlockHashAtDepth(int256 depth)

This method returns the serialized block header of a block identified by its depth.
A depth of 0 means the last block of the blockchain.
If the depth surpasses the initial block height, an empty byte array is returned.
The cost of this method call is 20K gas.

None of these new methods can be called onchain.


# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 

# Test Cases

TBD

## Security Considerations

This RSKIP keeps the methods inaccesible to onchain contracts. If they are opened to onchain calls, care must be taken to make sure the gas costs are fair, preventing DoS attacks.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).