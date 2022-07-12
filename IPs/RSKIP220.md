---
rskip: 220
title: Obtain Bitcoin Block information from bridge methods 
description: 
status: Adopted
purpose: Usa
author: PGP <pamela@iovlabs.org>, SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2021-03-26
---
# Obtain Bitcoin Block information from bridge methods

|RSKIP          |220           |
| :------------ |:-------------|
|**Title**      |Obtain Bitcoin Block information from bridge methods |
|**Created**    |26-MAR-21 |
|**Author**     |PGP,SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

## Abstract

The Bridge contract stores information about the Bitcoin blockchain. This RSKIP proposes the addition of new methods to the Bridge contract, that enable RSK smart contracts to query information about the Bitcoin best chain and retrieve its block headers. This introduces in RSK a decentralized cryptoeconomic (SPV) Bitcoin blockchain oracle for financial contracts, swaps, and other applications.

Also this proposal enables certain use cases where RSK contracts need cryptoeconomic random values. The header of a Bitcoin block contains cryptoeconomic randomness in its proof-of-work hash. A contract can wait until a certain Bitcoin block (specified by height) is added to the Bridge, and then use the header hash as a seed to extract pseudo-random values.

This change implies a consensus change and should be included in a network upgrade.

## Specification

The following new methods are added to the Bridge Contract:

### getBtcBlockchainBestBlockHeader

#### ABI signature

```
function getBtcBlockchainBestBlockHeader() returns (bytes)
```

#### Specification

This method simply returns the serialized bitcoin block header that is at the moment the tip of the mainchain.
The gas cost is 3800.

### getBtcBlockchainBlockHeaderByHash

#### ABI signature

```
function getBtcBlockchainBlockHeaderByHash(bytes32) returns (bytes)
```

#### Specification

This method expects to receive a Sha256 value representing a Bitcoin block header hash. If the hash matches a registered bitcoin block it will serialize the header and send it to the user, otherwise, it will return an empty byte array.
The gas cost is 4600.

### getBtcBlockchainBlockHeaderByHeight

#### ABI signature

```
function getBtcBlockchainBlockHeaderByHeight(uint256) returns (bytes)
```

#### Specification

This method expects to receive an integer value representing a bitcoin block height. The Bridge will lookup its storage for a block header on the mainchain matching the passed height. The height should be lower than the tip of the blockchain, otherwise the method will return an empty array.

This method relies on a block index to avoid expensive lookups. Heights lower than IRIS activation won't return blocks.
The gas cost is 5000.

### getBtcBlockchainParentBlockHeaderByHash

#### ABI signature

```
function getBtcBlockchainParentBlockHeaderByHash(bytes32) returns (bytes)
```

#### Specification

This methods expects a Sha256 value representing a bitcoin block hash. The Bridge will lookup the block the hash belongs to and later will get the parent of said block as the return value. For this method, if the parameterized block hash or the hash of its parent are not found, it will return an empty array.
The gas cost is 4900.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
