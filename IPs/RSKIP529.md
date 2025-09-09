---
rskip: 529
title: Super event storage entry
created: 26-AUG-25
author: MI
purpose: Sca
layer: Core
complexity: 1
status: Draft
description: 
---

|RSKIP          |529           |
| :------------ |:-------------|
|**Title**      |Super events storage entry |
|**Created**    |26-AUG-25 |
|**Author**     |MI |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

To be completed...

## Motivation

To be completed...

## Specification

### 1. New storage entry for the serialized `super event`

A new storage entry is necessary to store the serialized `super event` sent by the Union Bridge contract.

```
**Storage entry name:** `superEvent`

**Data type:** `bytes`

**Maximum size:** TBD
```

The value is considered *unset* if `superEvent.length == 0`.

### 2. Method to get the serialized `super event`

A method will be added to allow anyone to query the current `super event`. If no value is set, then it should return an empty array of bytes.

**Method signature:**

```
function getSuperEvent() public view returns (bytes memory);
```

### 3. Method to set the `super event` value in storage

A method will be added to allow the Union Bridge contract address to set the current `super event` value in storage. 

- If the caller is not the Union Bridge contract address then it should revert.
- It should receive a non-empty array of bytes and store it under the `superEvent` storage key of the Bridge contract. 
- If there is an already existing value under the `superEvent` storage key of the Bridge contract, it will be overriden by the new value.

**Method signature:**

```
function setSuperEvent(bytes memory superEvent) public;
```

### 4. Method to clear the `super event` value in storage

A method will be added to allow the Union Bridge contract address to clear the current `super event` value in storage. 

- If the caller is not the Union Bridge contract address then it should revert.
- It should store an empty byte array under the `superEvent` storage key of the Bridge, overriding whatever value was previously there.

**Method signature:**

```
function clearSuperEvent() public;
```

### 5. Method to get the accumulated difficulty of the latest `super block`

A method will be added to allow anyone to query the latest `super block` accumulated difficulty.

**Method signature:**

```
function getSuperBlockCumulativeDifficulty() public view returns(bytes);
```

### 6. Method to get the latest `super block` chain height

A method will be added to allow anyone to query the latest `super block` chain height.

**Method signature:**

```
function getSuperBlockchainHeight() public view returns(uint32);
```

## Rationale

Discuss design decisions, community debates and possible attacks.

## References

[1] [RSKIP502](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP502.md): PowPeg and Union Bridge integration  


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
