# Peg-in to any address

|RSKIP          |170           |
| :------------ |:-------------|
|**Title**      |Peg-in to any address |
|**Created**    |01-SEP-20 |
|**Author**     |MI |
|**Purpose**    |USa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes a way for users to be able to lock funds from BTC, indicating the RSK address in which they want their RBTCs transferred to and a BTC refund address in case the peg-in can not be completed for some reason.

## Motivation

Allowing users to lock funds to any address in RSK.

## Specification

The BTC transaction sent to the federation must contain at most one output with value 0 and OP_RETURN op code followed by the following data:
- Protocol version
- A valid RSK address (20 bytes)
- A valid BTC refund address (optional)

## Rationale

Discuss design decisions, community debates and possible attacks.

## References

[1] https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP41.md

[2] https://bitcoinfoundation.org/files/bitcoin/core-development-update-5/

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
