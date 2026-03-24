---
rskip: 552
title: Improve Blake2F Input Validation
created: 16-MAR-26
author: FML (@fmacleal)
purpose: Sec
layer: Core
complexity: 1
status: Draft
description: Add null-safety checks to Blake2F precompiled contract input handling
---

# Improve Blake2F Input Validation

|RSKIP          | 552                                          |
| :------------ |:---------------------------------------------|
|**Title**      | Improve Blake2F Input Validation              |
|**Created**    | 16-MAR-26                                    |
|**Author**     | FML                                          |
|**Purpose**    | Sec                                          |
|**Layer**      | Core                                         |
|**Complexity** | 1                                            |
|**Status**     | Draft                                        |

## Abstract

This RSKIP improves the input validation of the Blake2F precompiled contract (address `0x0000000000000000000000000000000000000009`) introduced by [RSKIP-153](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP153.md). The change ensures that the Blake2F precompile handles all possible calldata states correctly, including edge cases not covered by the original implementation. Additionally, the exception handling in the precompile execution path within the EVM is improved to increase robustness.

## Motivation

The Blake2F precompiled contract, enabled via RSKIP-153, expects calldata of exactly 213 bytes. The precompile validates the input length before proceeding. However, the current implementation does not handle all possible calldata edge cases consistently.

To ensure the Blake2F precompile handles all possible input states consistently and deterministically, explicit validation should be added for edge cases not covered by the original implementation. This aligns the precompile with defensive programming best practices and strengthens the overall robustness of the consensus execution path.

## Specification

This RSKIP introduces the following changes, activated conditionally via a new consensus rule (`RSKIP552`):

### 1. Improved input validation in gas calculation

When `RSKIP552` is active, the gas calculation logic for the Blake2F precompile handles all possible input states, including edge-case calldata. Invalid inputs return zero gas, consistent with the existing behavior for malformed input of incorrect length.

### 2. Improved input validation in execution

When `RSKIP552` is active, the execution logic for the Blake2F precompile handles all possible input states. Edge-case calldata is rejected with the existing error for incorrect input length, consistent with how other malformed inputs are handled.

### 3. Improved exception handling in precompile execution

The exception handling in the EVM's precompile execution path is improved to ensure that all error conditions during precompile execution are properly caught and handled, resulting in the call returning zero to the caller.

## Backward Compatibility

This change is activated via a consensus rule (`RSKIP552`) and will take effect at a specific network upgrade block height. Before activation, the behavior of the Blake2F precompile remains unchanged. After activation, the only behavioral difference is that transactions targeting the Blake2F precompile with edge-case calldata will be handled consistently with other malformed-input scenarios.

## Rationale

Precompiled contracts should handle all possible input states gracefully, following the principle of defensive input validation. This change ensures that the Blake2F precompile behaves consistently and deterministically regardless of how the calling transaction is encoded.

## References

[1] [RSKIP-153 - Add BLAKE2 Compression Function F Precompile](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP153.md)

[2] [EIP-152 - BLAKE2b F Compression Function](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-152.md)

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
