# Methods to Query the Redeem Scripts

|RSKIP          |xxx           |
| :------------ |:-------------|
|**Title**      |Methods to Query the Redeem Scripts |
|**Created**    |25-OCT-21 |
|**Author**     |SDL |
|**Purpose**    |Usa/Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

The Programmable Peg-in Addresses functionality (defined in [RSKIP176](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP276.md)) enables users to send bitcoins to RSK while calling user-specified contracts. This functionality was activated in Iris network upgrade. To call a user-specified contract, the target contract address, calling arguments, gas limit and other arguments are embedded in the Bitcoin address used for peg-in.  This new address is derived from the standard RSK peg-in address. The derivation method involves adding a prefix to the Powpeg P2SH redeem script. However, the Bridge contract does not expose any method to query the standard redeem script, or to compute a derived one. Therefore, to create a derived redeem script, the user has to assume the content of certain script chunks, such as the emergency multisignature public keys. Other parts of the script, such as the pegnatories public keys, can be retrieved using existing Bridge methods. However, using hard-coded script chunks makes script building error-prone and risky, because those assumed chunks may change in the future, rendering the script building function capable of locking user funds in derived addresses unrecognized by RSK. This RSKIP proposes the addition to the Bridge contract two several methods to obtain valid redeem script. These methods would be called off-chain by safe PPA-enabled libraries.

## Motivation

Stated in the abstract section.

## Implementation

Two new Bridge methods are added to the Bridge contract with signature:

```java
function getP2SHRedeemScriptForPPA(uint256 UPI) public returns (bytes redeemScript);
```

This method returns the redeem script associated with the provided UPI (as defined in RSKIP176), for the current active federation. The call consumes 5000 gas.

```java
function getP2SHRedeemScript() public returns (bytes redeemScript);
```

This method returns the redeem script for the standard peg-in address, for the current federation. The call consumes 4500 gas.

## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
