# Encode bridge events in Solidity format

|RSKIP          |146           |
| :------------ |:-------------|
|**Title**      |Encode bridge events in Solidity format |
|**Created**    |25-OCT-19 |
|**Author**     |MI |
|**Purpose**    |USa |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes a new way to encode Bridge events in Solidity format rather than the current RLP encoding.

## Motivation

The Bridge precompiled contract has a number of events, those events are formatted using RLP, which renders them incompatible with the existing web3/solidity tooling (explorer included).

Converting them to a compatible format would help users to easily get the data from those events.

## Specification

Since there are no indexed params in Bridge events, there will be only one topic (event signature) and event params will be encoded in the data field.

I suggest creating a way to easily create a `org.ethereum.core.CallTransaction.Function` of an `Event` type, and then use this defined event to generate the properly encoded topic and data.

From `org.ethereum.core.CallTransaction.Function` class, `encodeArguments` method can be used to encode arguments in Solidity format and `encodeSignatureLong` method to encode topic as Keccac256 hash.

## References

[1] https://solidity.readthedocs.io/en/v0.5.3/abi-spec.html#events

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
