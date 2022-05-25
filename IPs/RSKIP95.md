---
rskip: 95
title: DELEGATECALL as an instruction set extension 
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2018
---
#  **DELEGATECALL as an instruction set extension**  

| RSKIP          | 95                             |
| :------------- | :----------------------------- |
| **Title**      | DELEGATECALL as an instruction set extension |
| **Created**    | 2018                           |
| **Author**     | SDL                            |
| **Purpose**    | Sca                            |
| **Layer**      | Core                           |
| **Complexity** | 2                              |
| **Status**     | Draft                          |

# Abstract

This RSKIP proposes to expand the capabilities of the DELEGATECALL opcode to enable the implementation of new RSK-defined opcodes without colliding with Ethereum ones. 

# Motivation

RSK community has proposed many new opcodes to be added to the RVM. Most of these opcodes have no Ethereum counterpart. Adding new non-Ethereum opcodes to the RVM has the drawback that they may collide with new but different opcodes added to Ethereum. Even if the RVM has versioning system to allow using both the EVM-compatibility mode or the RVM machine, other tools do not support this versioning system. For example Solidiy can only output Ethereum-compatible code. We propose to expand DELEGATECALL to enable new opcodes to be accessed. The advantage is that Solidity supports arbitrary arguments sent to DELEGATECALL using the "assemble" block, and Ethereum has no plan to use DELEGATECALL as a gateway to new opcodes. From the user perspective, these new DELEGATECALLs look like calls to pre-compiled contracts, but internally the calls are handled much more efficiently, and there is no "context switch".  We'll call these pseudo-contracts "internal". The cost of a internal DELEGATECALL will depend on the internal opcode called.

## Specification

Internal contracts will be numbered starting from 0x100. The argument "gas" must be zero. The argument "codeAddress" must be in the range 0x100 to 0xfff. The arguments inDataOffs/inDataSize/outDataOffs/outDataSize will be defined differently for each new internal opcode, but must be specified (and pushed) even if unused. The last instructions before the DELEGATECALL MUST be PUSH 0x00, PUSH 0x0nnn. This means that neither the gas nor the opcode identification address can be dynamically computed. If the prior instructions are not PUSH 0x0nnn, PUSH 0x00,  then the DELEGATECALL will execute as before. Therefore prior DELEGATECALL the code must contain the bytes: 0x61 0x0n 0xnn 0x60 0x00. Forcing zero gas and an explicit opcode specification allows static analyzers to detect if a certain internal opcode is being called.  The DELEGATECALL can return any value on the top of the stack, or may also return values into memory pointed by the outDataOffs/outDataSize arguments. 

## Sample Solidity Code

```
contract Test {
    function t() public {
        bool result;
        assembly {
            result := delegatecall(
                0,     // gas must be zero
                0x123, // sample internal instruction code
                0,
                0,       // Size of the input (in bytes)                
                0,
                0       // Output is ignored, therefore the output size is zero
            )
        }
    }
}
```




# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


