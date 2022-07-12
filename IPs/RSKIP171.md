---
rskip: 171
title: Clean EVM Internal Buffer in Call-like Opcodes
description: 
status: Accepted
purpose: Usa
author: FJ (@fedejinich)
layer: Core
complexity: 1
created: 2020-02-09
---
# Clean EVM Internal Buffer in Call-like Opcodes

|RSKIP          |171           |
| :------------ |:-------------|
|**Title**      |Clean EVM Internal Buffer in Call-like Opcodes |
|**Created**    |02-09-2020 |
|**Author**     |FJ |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Accepted |

## Abstract

The purpose of this RSKIP is to make RSK fully compatible with [EIP-211](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-211.md). `RETURNDATACOPY` and `RETURNDATASIZE` opcodes have already been implemented in previous RSKIPs, but still, there were some corner cases related to call-like opcodes that weren't implemented in previous network upgrades.

Now with this RSKIP we have a mechanism to allow returning arbitrary-length data inside the EVM ([EIP-211](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-211.md)).

## Motivation

Described at [EIP-211](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-211.md).

## Specification

This RSKIP will be enabled only if `block.number >= IRIS_HARD_FORK`. 

Any opcode that creates a new call frame will be mentioned as call-like opcode

Considered call-like opcodes:
- `OP_CALL`
- `OP_CALLCODE`
- `OP_DELEGATECALL`
- `OP_STATICCALL`
- Further call-like opcodes may be added in future RSKIPs

Since most of the parts of mentioned EIP have already been implemented, this proposal only points to implement this described behaviour:


> If the call-like opcode is executed but does not really instantiate a call frame (for example due to insufficient funds for a value transfer or if the called contract does not exist), the return data buffer is empty.

## Reproducing misbehaviour

```
    // Considering this contract invocation call at 0x471fd3ad3e9eeadeec4608b92d16ce6b500704cc
    PUSH1 0x10
    PUSH1 0x05
    ADD
    PUSH1 0x40
    MSTORE
    PUSH1 0x20
    PUSH1 0x40
    RETURN

    // And this EVM execution
    PUSH1 0x20                                          // return size is 32 bytes
    PUSH1 0x40                                          // on free memory pointer
    PUSH1 0x00                                          // no argument
    PUSH1 0x00                                          // no argument size
    PUSH20 0x471fd3ad3e9eeadeec4608b92d16ce6b500704cc   // suppose a valid contract
    PUSH4 0x005B8D80                                    // with some gas
    STATICCALL                                          // call it (call-like opcode)! internal buffer should be 0x15
    PUSH1 0x20                                          // now do the same...
    PUSH1 0x40                                          // on free memory pointer
    PUSH1 0x00                                          // no argument
    PUSH1 0x00                                          // no argument size
    PUSH1 0x00                                          // but call a non-existent contract (this won't produce a new call frame)
    PUSH4 0x005B8D80                                    // with some gas
    STATICCALL                                          // call it (call-like opcode)!
```

Internal buffer should be 0x00 when RSKIP171 is enabled. If RSKIP171 is disabled and you execute any call-like opcode, the internal state buffer will remain the same, even if that execution didn't produce any new call frame. 

## Backwards Compatibility 

Apart from the described behaviour for call-like opcodes, this proposal stays fully backwards compatible.

## References

[1] EIP-211 https://github.com/ethereum/EIPs/blob/master/EIPS/eip-211.md

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
