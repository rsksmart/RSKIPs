# Emergency Time-locks Refresh


|RSKIP          | xxx |
| :------------ |:-------------|
|**Title**      |Emergency Time-locks Refresh|
|**Created**    |JAN-2021 |
|**Author**     | SDL |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

#  **Abstract**

The time-locked emergency multisignature introduced in RSKIP201 requires a that Powpeg UTXOs are periodically spent in order to prevent the time-lock expiration. 

This RSKIP proposes a mechanism for the Bridge to command this time-lock refresh in a way that maintains high-availability of funds for peg-out, and low risk of time-lock expiration.



## Motivation

TBD

## Specification

TBD

## Rationale

TBD

## Backwards Compatibility

TBD

## Implementation

RBD

## Security Considerations

This RSKIP protects the RSK network from the expiration of the time-lock. If the time-lock expired without a real block in the Powpeg devices, then the security of the pegged funds will relay on the the emergency multisignature signers, bypassing the security of the Powpeg.

While the community can force the pegged funds to be refreshed by performing a series of peg-outs, the blockchain should not rely on this behavior.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).