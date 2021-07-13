# Receive headers limits

|RSKIP          |200           |
| :------------ |:-------------|
|**Title**      |Receive headers limits |
|**Created**    |08-JAN-21 |
|**Author**     |PGP & MI |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes to make the current `receiveHeaders` bridge method private while creating a new public method `receiveHeader` that allows registering only one header at a time.

## Motivation

Bridge precompiled contract includes a function `receiveHeaders` that receives an array of headers to be stored. This function could be exposed to attacks.
Since powpeg nodes still need to send a group of headers to the Bridge, the current `receiveHeaders` function is made private. A new function `receiveHeader` is created, which only allows registering one header at a time, to be used by external users.

## Specification

To implement this change we suggest making the following modifications:

New function:
```
 reveive_header(bytes header) returns (int256 returnValue)
```

Make current `receive_headers` function private


### receive_header validations

1) The size of the parameter received is verified. These limits also exist in receive_headers function. 
2) Header is not previously saved in storage.
3) Time between calls to receive_headers is too short.
4) The previous block can not be found in storage. Block header received can not be connected with the previous one. 
5) The depth of these block headers is too old. The height of the block is compared with the maxDepthBlockchainAccepted. This value is also saved as a contact for each network. 

Every time that a block header is accepted, the timestamp is saved in BridgeStoragedProvider to can be compared the next time that receive_header is call. 

When the header was saved, the method returns 0. 
 
### receive_header return values
-  0  OK
- -1  RECEIVE_HEADER_CALLED_TOO_SOON
- -2  RECEIVE_HEADER_BLOCK_TOO_OLD
- -3  RECEIVE_HEADER_CANT_FOUND_PREVIOUS_BLOCK
- -4  RECEIVE_HEADER_BLOCK_PREVIOUSLY_SAVED
- -20 RECEIVE_HEADER_ERROR_SIZE_MISTMATCH
- -99 RECEIVE_HEADER_UNEXPECTED_EXCEPTION

### receive_headers changes

After the activation of this RSKIP, `receive_headers` can be called only by powpeg node.


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
