---
rskip: 119
title: Precompiled contract for inspecting block headers  
description: 
status: Draft
purpose: Usa
author: DM (@diego)
layer: Core
complexity: 1
created: 2019-04-01
---
# Precompiled contract for inspecting block headers

|RSKIP          |119 |
| :------------ |:------------- |
|**Title**      |Precompiled contract for inspecting block headers |
|**Created**    |01-APR-19 |
|**Author**     |DM |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

A new precompiled contract (i.e., native, hardwired onto the RSK consensus layer) is introduced. This contract contains operations to allow inspection of several block header fields that can be useful for smart contract developers.

## Motivation

Exploring the headers of specific blocks from solidity code (or assembly WOLOG) executing on the RVM could be too expensive. Doing this directly in solidity would require to write it and test it multiple times. Furthermore, different implementations can lead to errors potentially protruding to loss of funds. A well-written, tested and precompiled function set of operations to access the block header fields will tackle these problems while at the same time it will enforce certain level of security by implementing these as part of the RSK consensus.

## Specification
                                                    
A new precompiled contract will be accessible at the `0x0000000000000000000000000000000001000010` address. It will implement the following functions (signatures and return values are as follows):

- `getBitcoinHeader(uint256 blockDepth) returns (bytes)`
- `getBlockHash(uint256 blockDepth) returns (bytes)`
- `getCoinbaseAddress(uint256 blockDepth) returns (bytes)`
- `getGasLimit(uint256 blockDepth) returns (bytes)`
- `getGasUsed(uint256 blockDepth) returns (bytes)`
- `getMergedMiningTags(uint256 blockDepth) returns (bytes)`
- `getMinGasPrice(uint256 blockDepth) returns (bytes)`
- `getRSKDifficulty(uint256 blockDepth) returns (bytes)`
- `getUncleCoinbaseAddress(uint256 blockDepth, uint256 uncleIndex) returns (bytes)`

There is a limitation on how deep the block to inspect can be from the executing block. It is possible to recover fields of the past 4000 blocks, meaning that the blockDepth must be a value between 0 and 3999 inclusive, where blockDepth 0 references the parent block of the executing block, blockDepth 1 references the grandparent of the executing block and so on. If there is no block at the specified depth, an empty byte array is returned.

## Error handling

In case of error, each of these methods behaves as if a solidity `assert` statement was being used. That is, contract state is reverted and all the gas is consumed.

### getBitcoinHeader

The method `getBitcoinHeader(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query block header fields from and returns a byte array representation of the associated Bitcoin merged mining header.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 1,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the Bitcoin merged mining header for the parent block of the executing block. 

```
getBitcoinHeader(0)
```

We want to recover the Bitcoin merged mining header for the 2500th block starting at the executing block. 

```
getBitcoinHeader(2500)
```

We try to recover the Bitcoin merged mining header for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getBitcoinHeader(5000) => []
```

We try to recover the Bitcoin merged mining header for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getBitcoinHeader(3500) => []
```

### getBlockHash

The method `getBlockHash(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query block header fields from and returns a byte array representation of the block hash.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 1,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the block hash for the parent block of the executing block. 

```
getBlockHash(0)
```

We want to recover the block hash for the 2500th block starting at the executing block. 

```
getBlockHash(2500)
```

We try to recover the block hash for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getBlockHash(5000) => []
```

We try to recover the block hash for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getBlockHash(3500) => []
```

### getCoinbaseAddress

The method `getCoinbaseAddress(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query block header fields from and returns a byte array representation of the block's coinbase address.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 1,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the coinbase address for the parent block of the executing block. 

```
getCoinbaseAddress(0)
```

We want to recover the coinbase address for the 2500th block starting at the executing block. 

```
getCoinbaseAddress(2500)
```

We try to recover the coinbase address for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getCoinbaseAddress(5000) => []
```

We try to recover the coinbase address for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getCoinbaseAddress(3500) => []
```

### getGasLimit

The method `getGasLimit(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query block header fields from and returns a byte array representation of the block's gas limit.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 1,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the gas limit for the parent block of the executing block. 

```
getGasLimit(0)
```

We want to recover the gas limit for the 2500th block starting at the executing block. 

```
getGasLimit(2500)
```

We try to recover the gas limit for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getGasLimit(5000) => []
```

We try to recover the gas limit for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getGasLimit(3500) => []
```

### getGasUsed

The method `getGasUsed(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query block header fields from and returns a byte array representation of the block's gas used.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 1,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the gas used by the parent block of the executing block. 

```
getGasUsed(0)
```

We want to recover the gas used by the 2500th block starting at the executing block. 

```
getGasUsed(2500)
```

We try to recover the gas used by a block beyond the 4000 depth limit. An empty byte array is returned.

```
getGasUsed(5000) => []
```

We try to recover the gas used by a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getGasUsed(3500) => []
```

### getMergedMiningTags

The method `getMergedMiningTags(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query block header fields from and returns a byte array representation of the block's merged mining tags.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 1,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the merged mining tags for the parent block of the executing block. 

```
getMergedMiningTags(0)
```

We want to recover the merged mining tags for the 2500th block starting at the executing block. 

```
getMergedMiningTags(2500)
```

We try to recover the merged mining tags for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getMergedMiningTags(5000) => []
```

We try to recover the merged mining tags for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getMergedMiningTags(3500) => []
```

### getMinGasPrice

The method `getMinGasPrice(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query block header fields from and returns a byte array representation of the minimum gas price.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 1,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the minimum gas price for the parent block of the executing block. 

```
getMinGasPrice(0)
```

We want to recover the minimum gas price for the 2500th block starting at the executing block. 

```
getMinGasPrice(2500)
```

We try to recover the minimum gas price for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getMinGasPrice(5000) => []
```

We try to recover the minimum gas price for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getMinGasPrice(3500) => []
```

### getRSKDifficulty

The method `getRSKDifficulty(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query block header fields from and returns a byte array representation of the block's RSK difficulty.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 1,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the RSK's difficulty for the parent block of the executing block. 

```
getRSKDifficulty(0)
```

We want to recover the RSK's difficulty for the 2500th block starting at the executing block. 

```
getRSKDifficulty(2500)
```

We try to recover the RSK's difficulty for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getRSKDifficulty(5000) => []
```

We try to recover the RSK's difficulty for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getRSKDifficulty(3500) => []
```

### getUncleCoinbaseAddress

The method `getUncleCoinbaseAddress(uint256 blockDepth, uint256 uncleIndex) returns (bytes)` takes as input the depth of the block to query block header fields from, the index of the uncle to query for and returns a byte array representation of the coinbase address for the indexed uncle.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 1,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the coinbase address of the first uncle of the executing block's parent block. 

```
getUncleCoinbaseAddress(0, 0)
```

We want to recover the coinbase address of the third uncle of the 2500th block starting at the executing block. 

```
getUncleCoinbaseAddress(2500, 2)
```

We want to recover the coinbase address of the an unexistent uncle of the 2500th block starting at the executing block. An empty byte array is returned.

```
getUncleCoinbaseAddress(2500, 12) => []
```

We try to recover the coinbase address of an uncle for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getUncleCoinbaseAddress(5000, 0) => []
```

We try to recover the coinbase address of an uncle for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getUncleCoinbaseAddress(3500, 0) => []
```

## Rationale

TODO (if any)

## References

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
