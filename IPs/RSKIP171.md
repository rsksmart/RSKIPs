# Title

|RSKIP          |171           |
| :------------ |:-------------|
|**Title**      |Arbitrary-Length Data Return Mechanisim |
|**Created**    |02-09-2020 |
|**Author**     |FJ |
|**Purpose**    |USa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

The porpouse of this RSKIP is to be fully compatible with [EIP-211](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-211.md). `RETURNDATACOPY` and `RETURNDATASIZE` opcodes have already been implemented in previous RSKIPs, but still, there were some cases for call-like opcodes needed to be implemented.

Now with this RSKIP we have a mechanism to allow returning arbitrary-length data inside the EVM ([EIP-211](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-211.md)).

## Motivation

Described at [EIP-211](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-211.md)

## Specification

This RSKIP will be enabled only if `block.number >= IRIS_HARD_FORK`. 

Any opcode that creates a new call frame will be mentioned as call-like opcode

Considered call-like opcodes:
- `OP_CALL`
- `OP_CALLCODE`
- `OP_DELEGATECALL`
- `OP_STATICCALL`
- Further call-like opcodes may be added in future RSKIPs

Since most of the parts of mentioned EIP have already been implemented, this proporsal only points to implement this descrived behaviour:

```
If the call-like opcode is executed but does not really instantiate a call frame (for example due to insufficient funds for a value transfer or if the called contract does not exist), the return data buffer is empty.
```

## Backwards Compatibility 

Appart from descived behaviour for call-like opcodes, this proposal stays fully backwards compatible.

## References

[1] EIP-211 https://github.com/ethereum/EIPs/blob/master/EIPS/eip-211.md

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
