---
rskip: 438
title: Limit the maximum size of initcode and apply extra gas cost for every 32-byte chunk of initcode
description: 
status: Draft
purpose: Fair
author: FML (@fmacleal)
layer: Core
complexity: 2
created: 2024-07-16
---

# Limit the maximum size of initcode and apply extra gas cost for every 32-byte chunk of initcode

| RSKIP          | 437                                                                                             |
| :------------- |:------------------------------------------------------------------------------------------------|
| **Title**      | Limit the maximum size of initcode and apply extra gas cost for every 32-byte chunk of initcode |
| **Created**    | 16-JUL-2024                                                                                     |
| **Author**     | FML                   		                                                                        |
| **Purpose**    | Fair		                                                                                          |
| **Layer**      | Core                                                                                            |
| **Complexity** | 2                                                                                               |
| **Status**     | Draft                                                                                           |


The following RSKIP is an adaptation of [EIP3860](https://eips.ethereum.org/EIPS/eip-3860)

## Abstract
Extending the [EIP-170](https://eips.ethereum.org/EIPS/eip-170), that it's already implemented in RSK, this RSKIP proposes to limit the maximum size of the `initcode` 
(`MAX_INITCODE_SIZE =  2 * MAX_CONTRACT_SIZE = 49152`). Nevertheless, it's also proposed to introduce a charge of `2` gas  for every 32-byte chunk of `initcode`
to represent the cost of jumpdest-analysis.

## Motivation
During contract creation the client has to perform jumpdest-analysis on the `initcode` prior to execution. The work performed scales linearly with the size of the `initcode`. 
This work currently is not metered, nor is there a protocol enforced upper bound for the size.

There are three costs charged today:

1. Cost for `calldata` aka `initcode`: 4 gas for a byte with the value of zero, and 16 gas otherwise.
2. Cost for the resulting deployed code: 200 gas per byte.
3. Cost of address calculation (hashing of code) in case of `CREATE2` only: 6 gas per word.

Only the first cost applies to `initcode`, but only in the case of contract creation transactions. For the case of `CREATE`/`CREATE2` there is no such cost, and it is possible to programmatically 
generate variations of `initcode` in a relatively cheap manner.

## Specification

### Parameters
| **Constant**           | **Value**              |
|:-----------------------|:-----------------------|
| `INITCODE_WORST_COST`  | `2`                    |
| `MAX_INITCODE_SIZE`  | `2 * MAX_CONTRACT_SIZE` |

Where `MAX_CONTRACT_SIZE` is defined as `24576`.
We define `initcode_cost(initcode)` to equal `INITCODE_WORD_COST * ceil(len(initcode) / 32)`.

### Rules
1. If length of transaction data (`initcode`) in a create transaction exceeds `MAX_INITCODE_SIZE`, transaction is invalid.
> **Note**: This is similar to transactions considered invalid for not meeting the intrinsic gas cost requirement.)
2. For a create transaction, extend the transaction data cost formula to include` initcode_cost(initcode)`. 
> **Note**: This is included in transaction intrinsic cost, i.e. transaction with not enough gas to cover `initcode` cost is invalid.)
3. If length of `initcode` to `CREATE` or `CREATE2` instructions exceeds `MAX_INITCODE_SIZE`, instruction execution exceptionally aborts (as if it runs out of gas).
4. For the `CREATE` and `CREATE2` instructions charge an extra gas cost equaling to` initcode_cost(initcode)`. This cost is deducted before the calculation of the 
resulting contract address and the execution of `initcode`. 
> **Note**: This means before or at the same time as the hashing cost is applied in `CREATE2`.)

## Rationale

### Gas cost constant
The value of `INITCODE_WORD_COST` is selected based on performance benchmarks of differing worst-cases per implementation, more details about how this value was 
selected can be found in the [EIP-3860](https://eips.ethereum.org/EIPS/eip-3860#gas-cost-constant) that were based on to create this RSKIP.

### Gas cost per word (32-byte chunk)
The chosen cost of 2 gas per word was decided and discussed in the [EIP-3860](https://eips.ethereum.org/EIPS/eip-3860#gas-cost-per-word-32-byte-chunk). It's intended to
adopt the same values on this RSKIP. Therefore, the same implementation may be used for `CREATE` and `CREATE2` with different cost constants: before activation `0` for `CREATE` 
and `6` for `CREATE2`, after activation `2` for `CREATE` and` 6 + 2` for `CREATE2`.

### Reason for size limit of initcode
Estimating and creating worst case scenarios is easier with an upper bound in place, given one parameter for the search is greatly reduced. This allows for selecting a much more optimistic gas per byte.

Should there be no upper bound, the cost would need to be higher accounting for unknown unknowns. Given most initcode does not exceed the proposed limit, penalising contracts by overly conservative 
costs seems unnecessary.

### Effect of size limit of initcode
In most, if not all cases when a new contract is being created, the resulting runtime code is copied from the `initcode` itself. For the basic case the` 2 * MAX_CONTRACT_SIZE` limit allows `MAX_CONTRACT_SIZE` 
for runtime code and another `MAX_CONTRACT_SIZE` for contract constructor code. However, the limit may have practical implications for cases where multiple contracts are deployed in a single create transaction.

### Initcode cost for create transaction
The `initcode` cost for create transaction data `(0.0625` gas per byte) is negligible compared to the transaction data cost (4 or 16 gas per byte). 
Despite that, it was decided to include it in the specification for consistency, and more importantly for forward compatibility.

### How to report initcode limit violation?
It was specified that `initcode` size limit violation for `CREATE`/`CREATE2` results in exceptional abort of the execution. This places it in the group of early out-of-gas checks, 
including: stack underflow, memory expansion, static call violation, initcode hashing cost, and initcode cost introduced by this RSKIP. They precede the later “light” checks: call depth and balance.
The choice gives consistency to the order of checks and lowers implementation complexity (out-of-gas checks can be performed in any order).

## Backwards compatibility
This RSKIP requires a “network upgrade”, since it modifies consensus rules.

Already deployed contracts should not be effected, but certain transactions (with `initcode` beyond the proposed limit) would still be includable in a block, but result in an exceptional abort.

## Security Considerations
For client implementations, this RSKIP make attacks based on jumpdest-analysis less problematic. Increasing the robustness of clients.

For layer 2, this RSKIP introduces failure-modes where there previously were none. There could exist factory-contracts which deploy multi-level contract hierarchies, 
such that the code for multiple contracts are included in the `initcode` of the first contract. The author(s) of the [EIP-3860](https://eips.ethereum.org/EIPS/eip-3860#security-considerations) 
are not aware of any such contracts.

According to some analysis made on the EIP, with a `30M` gas limit, it would be possible to trigger jumpdest-analysis of a total `~1.3GB` of `initcode`. With this RSKIP, 
the cost of such an attack would be `~1.3GB * 2 / 32 = 81M` gas. 

## **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References
Base reference to write the RSKIP:

Martin Holst Swende (@holiman), Paweł Bylica (@chfast), Alex Beregszaszi (@axic), Andrei Maiboroda (@gumb0), "EIP-3860: Limit and meter initcode," Ethereum Improvement Proposals, no. 3860, July 2021. 
[Online serial]. Available: https://eips.ethereum.org/EIPS/eip-3860.