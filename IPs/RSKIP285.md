---
rskip: 285
title: Utility Methods to Make PPA Safer
description: 
status: Draft
purpose: Usa, Sec
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2021-10-25
---

# Utility Methods to Make PPA Safer

|RSKIP          |285           |
| :------------ |:-------------|
|**Title**      |Utility Methods to Make PPA Safer |
|**Created**    |25-OCT-21 |
|**Author**     |SDL |
|**Purpose**    |Usa/Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

The Programmable Peg-in Addresses functionality (defined in [RSKIP176](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP276.md)) enables users to send bitcoins to RSK while calling user-specified contracts. This functionality was activated in Iris network upgrade. To call a user-specified contract, the target contract address, calling arguments, gas limit and other arguments are embedded in the Bitcoin address used for peg-in.  This new address is derived from the standard RSK peg-in address. The derivation method involves adding a prefix to the Powpeg P2SH redeem script. However, the Bridge contract does not expose any method to query the standard redeem script, or to compute a derived one. Therefore, to create a derived redeem script, the user has to assume the content of certain script chunks, such as the emergency multisignature public keys. Other parts of the script, such as the pegnatories public keys, can be retrieved using existing Bridge methods. However, using hard-coded script chunks makes script building error-prone and risky, because those assumed chunks may change in the future, rendering the script building function capable of locking user funds in derived addresses unrecognized by RSK. This RSKIP proposes the addition to the Bridge contract of a method to obtain a valid redeem scripts. Also we add more utility methods to inspect the emegency multisig. The new methods would normally be called off-chain by safe PPA-enabled libraries.

## Motivation

Stated in the abstract section.

## Implementation

The following new methods are added to the Bridge contract:

```java
function getP2SHRedeemScriptForPPA(uint256 UPI) public returns (bytes redeemScript);
```

This method returns the redeem script associated with the provided UPI (as defined in RSKIP176), for the current active federation. The call consumes 5000 gas.

```java
function getP2SHAddressForPPA(uint256 UPI) public returns (bytes21 address);
```
This method returns a binary representation of the Bitcoin P2SH address associated with the UPI. The representation consist of a 1-byte version followed by a RIPEMD160 hash. The caller can build the ASCII representation of the Bitcoin address from the binary representation.
The call consumes 5500 gas.

```java
function getP2SHRedeemScript() public returns (bytes redeemScript);
```

This method returns the redeem script for the standard peg-in address, for the current federation. The call consumes 4500 gas.

```java
function getEmergencyPublicKeyOfType(int256 index, string type) public returns (bytes pubkey);
```
This method returns a public key of the emergency multisig.
Currently only type "btc" is supported. In the future the emergency multisig may be provided by a backup Powpeg and the remaining public key types may be enabled. Indicating an invalid type or index generates a OOG exception. The call consumes 3000 gas.

```java
function getEmergencyMultisigSize() public returns (int256 size);
```
This method returns the number of parties in the emergency multisig. Returns 0 if the emergency multisig is deactivated. The call consumes 3000 gas.

```java
function getEmergencyMultisigThreshold() public returns (int256 thr);
```
This method returns the number of signatures required to access the pegged funds in case of emergency. The call consumes 3000 gas.



## Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
