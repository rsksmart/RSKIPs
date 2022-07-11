---
rskip: 336
title: Simple Parallelizable Semaphore
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2022-07-11
---
# Simple Parallelizable Semaphore

|RSKIP          |336           |
| :------------ |:-------------|
|**Title**      |Simple Parallelizable Semaphore|
|**Created**    |11-JUL-2022 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     ||

# **Abstract**

This RSKIP proposes adding a precompiled contract that provides a semaphore function. This function aims to replace the typical semaphores implemented as contract  storage cells. The benefit is that the precompile-based semaphore does not write to storage, and therefore is is parallelizable under [RSKIP-144](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP144.md), while the typical sempaphore is not. 

# **Motivation**

The typical smart contract semaphore uses a contract storage cell. The algorithm is simple: the code checks that a storage cell is zero on entry, aborting if not, and then sets it to 1. After executing the required code, it resets the cell back to 0 before exiting. This is the algorithm implemented in OpenZeppelin's [ReentrancyGuard](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/ReentrancyGuard.sol). The algorithm results in a read-write pattern on the semaphore's storage cell. This pattern prevents the parallelization of the execution of the smart contract using [RSKIP-144](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP144.md).
The only way to paralellize contracts using this semaphore contruction would be that the VM automatically detect that a variable is used as a sempahore, and proves that the sempahore behaves as required by static analysys. This is difficult for two reasons. First, the CPU cost of detection and/or proving may be high. Second, some contract functions may not be protected by the semaphore, meaning that some execution paths do not alter the semaphore, which may complicate proving. 
Therefore this proposal aims to protect future contracts, rather than to protect already deployed ones.

# **Specification**

Starting from an activation block (TBD) a new precompiled contract `Semaphore` is created at address 0x01000011. when `Semaphore` is called, if the caller address is present more than once in the stack, the contract behaves as if the first instruction had been a REVERT, therefore the CALL returns 0. Otherwise, it executes no code and returns 1. The gas cost of the contract execution is set to 500, which is consumed independently of the call result.

# Rationale

There are alternative designs to implement semaphores on the EVM such as 
[EIP-1153: Transient storage opcodes](https://eips.ethereum.org/EIPS/eip-1153) or [`SSTORE_COUNT`](https://github.com/ethereum/EIPs/issues/119). The first does reduce the cost of using semaphores, but does not address the problem of parallelization. The second enables execution parallelization but it is more complex as it returns the number `SSTORE` opcodes executed, not the number of reentrant calls. Reentrancy must be deducted from this value. Also the count `SSTORE` count is not currently tracked by the RSK EVM. Also, it is discuraged to implement a new opcode that does not exists in Ethereum as it increases the chances of future incompatibilities. The presented solution is cleaner and simpler than all other alternatives.

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients do not need to be updated. 
Applications using reentrancy semaphores that are ported from Ethereum to RSK that want to take advantage of RSK parallelization would need to replace the storage sempahore by a call to the `Semaphore` contract. This change can be easily encapsulated, such as modifying the implementation of nonReentrant function modifier. It is also possible to program a contract code that uses a storage semaphore or the `Semaphore` contract depending on the chainid. Deployed contracts and contracts that are ported unmodified from Ethereum will continue to work as expected after this change.

# Test Cases

TBD

## Security Considerations

No security risks have been identified related to this change.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
