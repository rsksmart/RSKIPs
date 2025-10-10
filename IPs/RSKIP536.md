---
rskip: 536
title: Additional methods for BlockHeader precompiled contract
description: 
status: Draft
purpose: Usa
author: MI
layer: Core
complexity: 1
created: 2025-10-10
---
# Additional methods for BlockHeader precompiled contract

|RSKIP          |536 |
| :------------ |:------------- |
|**Title**      |Additional methods for BlockHeader precompiled contract |
|**Created**    |10-OCT-25 |
|**Author**     |MI |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP extends the BlockHeader precompiled contract [[1]](#references) by adding two new methods to retrieve difficulty-related information from block headers: `getTotalDifficulty` and `getCumulativeDifficulty`. These methods provide access to the total difficulty and cumulative difficulty values for blocks within the accessible depth range.

## Motivation

The existing BlockHeader precompiled contract provides access to various block header fields, including RSK difficulty through the `getRSKDifficulty` method. However, it lacks access to total difficulty and cumulative difficulty values.

## Specification

This RSKIP adds two new methods to the existing BlockHeader precompiled contract at address `0x0000000000000000000000000000000001000010`:

- `getTotalDifficulty(uint256 blockDepth) returns (bytes)`
- `getCumulativeDifficulty(uint256 blockDepth) returns (bytes)`

Both methods follow the same validation rules and gas cost structure as the existing methods specified in RSKIP119 [[1]](#references).

### Block Depth Limitation

The same limitation applies as defined in RSKIP119: it is possible to recover fields of the past 4000 blocks, meaning that the `blockDepth` must be a value between 0 and 3999 inclusive, where:
- `blockDepth 0` references the parent block of the executing block
- `blockDepth 1` references the grandparent of the executing block
- And so on...

If there is no block at the specified depth, an empty byte array is returned.

### Error Handling

In case of error, each of these methods behaves as if a solidity `assert` statement was being used. That is, contract state is reverted and all the gas is consumed.

### getTotalDifficulty

The method `getTotalDifficulty(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query and returns a byte array representation of the total cumulative difficulty of the chain up to that block.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 4,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the total difficulty for the parent block of the executing block.

```
getTotalDifficulty(0)
```

We want to recover the total difficulty for the 2500th block starting at the executing block.

```
getTotalDifficulty(2500)
```

We try to recover the total difficulty for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getTotalDifficulty(5000) => []
```

We try to recover the total difficulty for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getTotalDifficulty(3500) => []
```

### getCumulativeDifficulty

The method `getCumulativeDifficulty(uint256 blockDepth) returns (bytes)` takes as input the depth of the block to query and returns a byte array representation of the cumulative difficulty for that block. This is the difficulty of the block itself plus its uncles.

#### Validations

- `blockDepth` must be an unsigned value ranging from 0 to 3999.

#### Gas cost

This method has a fixed cost of 4,000 gas units. On top of that, normal transaction gas costs apply.

#### Sample usage

We want to recover the cumulative difficulty for the parent block of the executing block.

```
getCumulativeDifficulty(0)
```

We want to recover the cumulative difficulty for the 2500th block starting at the executing block.

```
getCumulativeDifficulty(2500)
```

We try to recover the cumulative difficulty for a block beyond the 4000 depth limit. An empty byte array is returned.

```
getCumulativeDifficulty(5000) => []
```

We try to recover the cumulative difficulty for a block that is not beyond the 4000 depth limit but still does not exist. An empty byte array is returned.

```
getCumulativeDifficulty(3500) => []
```

## Rationale

The design decisions for this RSKIP are based on the following considerations:

1. **Consistency with existing methods**: Both new methods follow the same pattern, validation rules, and gas costs as the existing methods in RSKIP119, ensuring a consistent API.

2. **Gas cost**: The fixed cost of 4,000 gas units is consistent with other methods in the BlockHeader precompiled contract, providing predictable gas consumption.

3. **Return format**: Both methods return `bytes` to maintain consistency with the existing API and provide flexibility for different data representations.

4. **Error handling**: Using the same error handling mechanism (assert-like behavior) as existing methods ensures consistent behavior across the precompiled contract.

5. **Block depth limitation**: Maintaining the same 4000-block depth limitation ensures consistency with existing methods and prevents excessive resource usage.

The addition of these methods provides essential functionality for smart contracts that need to analyze network security metrics and mining activity, while maintaining the efficiency and reliability of the precompiled contract approach.

## References

[1] RSKIP119: Precompiled contract for inspecting block headers https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP119.md

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
