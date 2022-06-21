# Add method `getActivePowpegRedeemScript` to the Bridge contract and perform additional Flyover peg-in validations

|RSKIP          |293           |
| :------------ |:-------------|
|**Title**      |Add method `getActivePowpegRedeemScript` to the Bridge contract and perform additional Flyover peg-in validations |
|**Created**    |18-JAN-22 |
|**Author**     |JD |
|**Purpose**    |Usa,Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Accepted |

## Abstract

Flyover protocol was added with RSKIP176 and included in the Blockchain since Iris consensus change.
This protocol will enhance the usability of the peg-in protocol to enable users to get RBTC faster and even execute smart contracts by sending BTC to a specific POWpeg address.

This RSKIP proposes improvements to the underlying protocol adding functionality that will improve its usability.

## Motivation

Since the protocol was launched there were reports of issues that needs to be addressed in order for this protocol to truly add value to the community.

## Specification

This RSKIP proposes fixing technical debts and also increasing the autonomy of the consumers at the moment of validate the information the Liquidity Provider sends.

### Considering the retiring POWpeg when processing flyover peg-ins

The Bridge will verify if there is a retiring POWpeg in existence while processing the peg-in. If there is, it will look for UTXOs to this address as well.

The found UTXOs will be stored in the corresponding bucket, waiting for the migration to the active POWpeg.

This UTXO processing undergoes the regular verifications (e.g. value, existance in the block, etc).

### Checking the minimum value for peg-ins is respected in each UTXO

The Bridge will check that the flyover peg-ins respect the minimum values as the regular peg-in, enforcing it by each UTXO to the Flyover POWpeg address.

Any value below the minimum for peg-in will generate a new error code: `-305`.

### Adding new method to the Bridge to share the redeemscript of the active POWpeg

#### signature

```
function getActivePowpegRedeemScript() returns (bytes)
```

#### Definition

This new method is a public method that will return the redeems cript of the active POWpeg. This could be a regular multisig redeem script, or it could be an ERP redeem script (See RSKIP201).

The method should take the active POWpeg information from the storage and serialize the redeem script to send it as a response to this method.

This method should fail if it is called before the consensus change activates.

## Rationale

There are no attack vectors detected for the proposed improvements.


## References

[1](RSKIP176.md) Flyover RSKIP
[2](RSKIP201.md) Time-locked Emergency Multisignature RSKIP

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
