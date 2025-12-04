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
|**Title**      |New storage cells in Bridge native contract for base and super events info |
|**Created**    |26-AUG-25 |
|**Author**     |MI |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

This RSKIP adds new storage cells to the Bridge native contract for storing base and super event information. These storage entries allow the Union Bridge contract to store peg-out event data that can be used for zero-knowledge proofs of cumulative work of the Rootstock chain, enabling more efficient verification of specific peg-out events at a much lower cost than retrieving data from contract storage.

## Motivation

RSKIP535 [[2]](#references) defines how base event information can be stored in the block header extension to enable efficient zero-knowledge proofs of cumulative work. However, for this mechanism to work effectively, the Union Bridge contract needs a way to store and manage the current base and super event data that will be included in block headers.

The Bridge native contract provides persistent storage capabilities to maintain the current state of base and super events. As described in RSKIP535 [[2]](#references), this data is then used by miners to populate the baseEvent field in block headers, enabling the zero-knowledge proof functionality.

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
- It should receive an array of bytes whose length is at most 128 bytes, which it then stores under the `baseEvent` storage key of the Bridge contract. 
- If there is an already existing value under the `baseEvent` storage key of the Bridge contract, it will be overridden by the new value.

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
- It should receive an array of bytes whose length is at most 128 bytes, which it then stores under the `superEvent` storage key of the Bridge contract. 
- If there is an already existing value under the `superEvent` storage key of the Bridge contract, it will be overridden by the new value.

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

The design decisions for this RSKIP are based on the following considerations:

### Storage Design
**128-byte size limitation**: This limit prevents excessive storage usage while accommodating typical peg-out event data. It also prevents potential DoS attacks through large data storage and keeps gas costs predictable.

### Access Control
**Union Bridge contract-only writes**: Restricting write access to the Union Bridge contract address ensures data integrity and prevents unauthorized modifications. This follows the principle of least privilege and maintains the security model of the bridge system.

**Public read access**: Allowing anyone to read the event data enables transparency and supports the zero-knowledge proof mechanisms described in RSKIP535.

### Backwards Compatibility
**Non-intrusive design**: The new storage cells do not interfere with existing Bridge contract functionality, ensuring smooth upgrades and maintaining existing bridge operations.

**Optional usage**: The storage cells can be used independently, allowing for gradual adoption and testing without breaking existing systems.

## References

[1] [RSKIP502](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP502.md): PowPeg and Union Bridge integration  

[2] [RSKIP535](https://github.com/rsksmart/RSKIPs/blob/master/IPs/RSKIP535.md): Add the baseEvent field to the Block header extension  


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
