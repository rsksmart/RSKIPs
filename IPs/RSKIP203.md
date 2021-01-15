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

This RSKIP proposes to add a method `getCallStackDepth()` to the BlockHeader pre-compile contract that retrieves the current depth of the EVM call stack.

## Motivation

There are many smart-contract systems that need to receive commands for execution of a user-defined contract, where the command is forwarded by a third-party that acts as a relayer and a sponsor providing certain value in advance, and the value is reimbursed to the sponsor if the transaction was correctly executed. This happens in meta-transaction systems and in the fast BTC bridge proposal for RSK. An execution is correct if all the environment parameters agreed by the user and sponsor are replicated. For example, the contract to call, the call arguments and the gas limit. However, in RSK,  current stack depth is an environment parameter that a contract cannot fully verify. The correctness of execution also depends on it, as calling a contract with low call stack space can result in a completely different execution outcome due to failing nested calls. The problem was identified in Ethereum early on, and [EIP-3](https://eips.ethereum.org/EIPS/eip-3) (one of the first EIPs) proposes a `CALLDEPTH` opcode to solve the problem. However the EIP was not approved, and when Ethereum adopted EIP150, it also adopted an infinite stack, so the problem cannot occur in Ethereum. However, RSK did not adopt the 63/64 gas reduction rule and therefore the problem may persist in RSK, although as Solidity checks every call return code, the problem affects only contracts that forward calls relayed by sponsors.

The current solution in RSK is to check if the call stack is empty or non-empty, which is possible in the EVM. Contracts can require that the transaction origin matches the transaction sender. This check works because an origin account is always an Externally Owned Account (EOA) and as EOAs have no code, they cannot participate in nested calls. However, this solution is suboptimal for two reasons. First, it prevents the relayer from performing batch executions. To achieve  batch execution, the relayer packs several calldatas from different users in a single external transaction, reducing transaction costs. To sequence the execution, an intermediate smart-contract must be used, so the origin check fails. It's possible that in the future code can be executed directly in the EOC context but not permanently installed, such as proposed by the [rich-transactions ](https://github.com/Arachnid/EIPs/blob/richtx/EIPS/EIP-draft-rich-transactions.md) but this option is not currently available. Second, the origin check breaks if in the future RSK adopts account abstraction enabling EOA to have code permanently installed, such as the Install [Code Precompile proposal](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP167.md).

This RSKIP proposes to add to the BlockHeaders precompile a method to obtain the call stack depth. The user and sponsor can agree on a minimum call stack depth that enables both batching and ample stack space for the user call.

## Specification

The method with signature `GetCallStackDepth() returns (uint32)` is added to the BlockHeader contract that exists at address `0x0000000000000000000000000000000001000010`. The call can fail is the call stack is full. The minimum returned value is 1, because the call to the precompile contract consumes one call stack position.


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

 
