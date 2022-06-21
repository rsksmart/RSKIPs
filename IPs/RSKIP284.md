# New deserialization method for Flyover refund addresses

|RSKIP          |284           |
| :------------ |:-------------|
|**Title**      |New deserialization method for Flyover refund addresses |
|**Created**    |19-OCT-21 |
|**Author**     |JD |
|**Purpose**    |USa, Sec|
|**Layer**      |Core |
|**Complexity** |1|
|**Status**     |Draft|

## Abstract

Flyover protocol was added with RSKIP176 and included in the Blockchain since Iris consensus change.
This protocol will enhance the usability of the peg-in protocol to enable users to get RBTC faster and even execute smart contracts by sending BTC to a specific POWpeg address.

This RSKIP proposes improvements to the underlying protocol without making changes to its usage.

## Motivation

Since the protocol was launched there were reports of issues that needs to be addressed in order for this protocol to truly add value to the community.

## Specification

This RSKIP proposes fixing one of the reported issues that rendered the service unusable.

### Fix Bitcoin refund addresses deserialization

The protocol includes the requirement to define a Bitcoin refund address for each of the actors (i.e. user and Liquidity Provider). Each of these addresses are serialized using a byte for the version and the hash160 representation of the public key / script. This was done in order to reduce the need for bytes for each address.

Reviewing the existing implementation it was detected that the consensus was invalid as it was using a BigInteger data type to decode the version. BigInteger uses the least significant bit to represent the sign. This means that any byte starting with a 1 will be automatically converted to a negative value.
This happens in fact, when the version is p2sh in Testnet, in that case the version byte is C4, which is interpreted by BigInteger as -195 (the binary representation of C4 is 11000100 and BigInteger takes the first bit as the signum leading to reinterpretate the value as 01000100).

On top of this, it was detected that the protocol is not enforcing the length of the data corresponding to the address. With this a compromised wallet could manipulate the UX to show certain data to the user but send additional data to the protocol and it would accept it. The Bridge should enforce the data length to be 21 bytes, no more nor less.

### Proposed solution

The RSKIP proposes changing the deserialization to instead of relying on BigInteger to manually read the first byte as the Version and the following 20 bytes as the Hash160 represensation of the script / public key.
This, combined with the network parameters already available in the Bridge, is enough to properly parse the refund address to a valid base58 address. It is important to remind the reader that ATM the moment the Bridge can only refund base58 addresses.

Additionally, any data length different than 21 bytes should be rejected.


## References

[1](RSKIP176.md) Flyover RSKIP

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
