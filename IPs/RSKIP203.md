---
rskip: 203
title: getCallStackDepth Precompile method
description: 
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2021-01
---
# getCallStackDepth Precompile method


|RSKIP          | 203 |
| :------------ |:-------------|
|**Title**      |getCallStackDepth Precompile method|
|**Created**    |JAN-2021 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |


# **Abstract**

This RSKIP proposes to add a method `getCallStackDepth()` to a new `Environment`  pre-compile contract. The new method retrieves the current depth of the EVM call stack.

## Motivation

Several smart-contract systems involve situations where end users' commands are received indirectly. A command is sent by a third-party (which serves as a *relayer*) to a *management contract*, which performs some accounting and then forwards it to a *destination contract*. In some designs, an entity called a *sponsor* provides certain value in advance on behalf of the end user. The value is used to pay for gas or to advance a payment to the destination contract. The role of the management contract is to reimburse the value to the sponsor if the transaction was correctly executed. This architecture is used by some meta-transaction systems and in the fast BTC bridge proposal for RSK. The call is correctly executed if all the environment parameters agreed by the user and sponsor are replicated. For example, the contract to call, the call arguments and the gas limit. However, in RSK,  current stack depth is an environment parameter that a contract cannot fully verify. The correctness of execution also depends on it, as calling a contract with low call stack space can result in a completely different execution outcome due to failing nested calls. The problem was identified in Ethereum early on, and [EIP-3](https://eips.ethereum.org/EIPS/eip-3) (one of the first EIPs) proposes a `CALLDEPTH` opcode to solve the problem. However the EIP was not approved, and when Ethereum adopted EIP150, it also adopted an infinite stack, so the problem cannot occur in Ethereum. However, RSK did not adopt the 1/64 gas reduction rule and therefore the problem may persist in RSK, although it only affects contracts that forward calls relayed by sponsors. The reason other contracts are not affected is because since 2016 Solidity emits a warning if a contracts uses the call() instruction without checking the return value. For all the remaining calls executed using contract methods, Solidity automatically adds code to check for the return value, and rises an OOG exception when the a call fails, forcing execution to abort with a failure code, and OOG exceptions will be thrown in cascade until the outermost call returns a failure code. But this failure code cannot be used by forwarders,  because the sponsor must be economically compensated even if the call fails with OOG. 

The current solution in RSK is to check if the call stack is empty or non-empty, which is possible in the EVM. Contracts can require that the transaction origin matches the transaction sender. This check works because an origin account is always an Externally Owned Account (EOA) and as EOAs have no code, they cannot participate in nested calls. However, this solution is suboptimal for two reasons. First, it prevents the relayer from performing batch executions. To achieve  batch execution, the relayer packs several calldatas from different users in a single external transaction, reducing transaction costs. To sequence the execution, an intermediate smart-contract must be used, so the origin check fails. It's possible that in the future code can be executed directly in the EOA context but not permanently installed, such as proposed by the [rich-transactions ](https://github.com/Arachnid/EIPs/blob/richtx/EIPS/EIP-draft-rich-transactions.md) but this option is not currently available. Second, the origin check breaks if in the future RSK adopts account abstraction enabling EOA to have code permanently installed, such as the Install [Code Precompile proposal](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP167.md).

This RSKIP proposes to create a new `Environment` pre-compile contract to with a method to obtain the call stack depth. Using this method, the user and sponsor can agree on a minimum call stack depth that can be checked by the contract code before execution begins. The agreed stack depth would enable batching and give ample stack space for the user call.

## Specification

The method with signature `GetCallStackDepth() returns (uint32)` is added to a new pre-compiled contract named `Environment`, created at address `0x0000000000000000000000000000000001000011`. The call can fail is the call stack is full. The minimum returned value is 1, because the call to the precompile contract consumes one call stack position.

The gas cost specific to this method call is 0, but the existing 700 gas for any contract call will be charged.


## Backwards Compatibility

This is a hard-forking change, but it shouldn't affect existing smart contracts, because no contract should call a precompile contract with an nonexistent method selector.


## Test Cases

TBD

## Implementation

TBA

## Security Considerations

None identified.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).,

 
