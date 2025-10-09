---
rskip: 529
title: New storage cells in Bridge native contract for base and super events info
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
|**Title**      |Super events |
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

### 1. Base event

#### 1.1 New storage entry for the serialized `base event`

A new storage entry is necessary to store the serialized `base event` sent by the Union Bridge contract.

```
**Storage entry name:** `baseEvent`

**Data type:** `bytes`

**Maximum size:** 128 bytes
```

#### 1.2. Method to get the serialized `base event`

A method will be added to allow anyone to query the current `base event`. If no value is set, then it should return an empty array of bytes.

**Method signature:**

```
function getBaseEvent() public view returns (bytes memory);
```

#### 1.3. Method to set the `base event` value in storage

A method will be added to allow the Union Bridge contract address to set the current `base event` value in storage. 

- If the caller is not the Union Bridge contract address then it should revert.
- It should receive a non-empty array of bytes whose length is at most 128 bytes, which it then stores under the `baseEvent` storage key of the Bridge contract. 
- If there is an already existing value under the `baseEvent` storage key of the Bridge contract, it will be overriden by the new value.

**Method signature:**

```
function setBaseEvent(bytes memory baseEvent) public;
```

#### 1.4. Method to clear the `base event` value in storage

A method will be added to allow the Union Bridge contract address to clear the current `base event` value in storage. 

- If the caller is not the Union Bridge contract address then it should revert.
- It should store an empty byte array under the `baseEvent` storage key of the Bridge, overriding whatever value was previously there.

**Method signature:**

```
function clearBaseEvent() public;
```

### 2. Super event

#### 2.1 New storage entry for the serialized `super event`

A new storage entry is necessary to store the serialized `super event` sent by the Union Bridge contract.

```
**Storage entry name:** `superEvent`

**Data type:** `bytes`

**Maximum size:** 128 bytes
```

#### 2.2. Method to get the serialized `super event`

A method will be added to allow anyone to query the current `super event`. If no value is set, then it should return an empty array of bytes.

**Method signature:**

```
function getSuperEvent() public view returns (bytes memory);
```

#### 2.3. Method to set the `super event` value in storage

A method will be added to allow the Union Bridge contract address to set the current `super event` value in storage. 

- If the caller is not the Union Bridge contract address then it should revert.
- It should receive a non-empty array of bytes whose length is at most 128 bytes, which it then stores under the `superEvent` storage key of the Bridge contract. 
- If there is an already existing value under the `superEvent` storage key of the Bridge contract, it will be overriden by the new value.

**Method signature:**

```
function setSuperEvent(bytes memory superEvent) public;
```

#### 2.4. Method to clear the `super event` value in storage

A method will be added to allow the Union Bridge contract address to clear the current `super event` value in storage. 

- If the caller is not the Union Bridge contract address then it should revert.
- It should store an empty byte array under the `superEvent` storage key of the Bridge, overriding whatever value was previously there.

**Method signature:**

```
function clearSuperEvent() public;
```

## Rationale

Discuss design decisions, community debates and possible attacks.

## References

[1] [RSKIP502](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP502.md): PowPeg and Union Bridge integration  


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
