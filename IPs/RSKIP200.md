---
rskip: 200
title: Receive headers limits
description: 
status: Adopted
purpose: Sec
author: PGP <pamela@iovlabs.org>, MI <marcos@iovlabs.org>
layer: Core
complexity: 2
created: 2021-01-08
---
# Receive headers limits

|RSKIP          |200           |
| :------------ |:-------------|
|**Title**      |Receive headers limits |
|**Created**    |08-JAN-21 |
|**Author**     |PGP & MI |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Adopted |

## Abstract

This RSKIP proposes to make the current `receiveHeaders` bridge method private while creating a new public method `receiveHeader` that allows registering only one header at a time.

## Motivation

Bridge precompiled contract includes a function `receiveHeaders` that receives an array of headers to be stored. This function could be exposed to attacks.
Since powpeg nodes still need to send a group of headers to the Bridge, the current `receiveHeaders` function is made private. A new function `receiveHeader` is created, which only allows registering one header at a time, to be used by external users.

## Specification

To implement this change we suggest making the following modifications:

1. Create a new method in the bridge: 
    ```
    receiveHeader(bytes header) returns (int256 returnValue)
    ```

2. Make current `receiveHeaders` function private


### `receiveHeader` validations

1) Header has expected size. 
2) Header is not previously saved in storage.
3) Time between calls to receiveHeader is larger than the minimum time between calls allowed. This value can be configured for each network. Initially the value is 300 seconds for all networks.
4) Header previous block must be found in storage.
5) Block depth of the received header is not above the maximum depth allowed. This value can be configured for each network, but initially this is 25 blocks for all networks. 

Every time that a block header is accepted, the timestamp is saved in storage to validate that no more calls are made to `receiveHeader` until the minimum configured time has elapsed.

When the header is successfully saved, the method returns the value `0`. 
 
### `receiveHeader` return values
-  0  OK
- -1  RECEIVE_HEADER_CALLED_TOO_SOON
- -2  RECEIVE_HEADER_BLOCK_TOO_OLD
- -3  RECEIVE_HEADER_CANT_FOUND_PREVIOUS_BLOCK
- -4  RECEIVE_HEADER_BLOCK_PREVIOUSLY_SAVED
- -20 RECEIVE_HEADER_ERROR_SIZE_MISTMATCH
- -99 RECEIVE_HEADER_UNEXPECTED_EXCEPTION

### `receiveHeaders` changes

After the activation of this RSKIP, `receiveHeaders` can be called only by members of the Federation.


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
