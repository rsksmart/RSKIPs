# Bridge performance improvement

|RSKIP          |200           |
| :------------ |:-------------|
|**Title**      |Receive headers limits |
|**Created**    |08-JAN-21 |
|**Author**     |PGP |
|**Purpose**    |Sca,USa,Sec |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

## Abstract

This RSKIP proposes to include new functionality in the bridge precompiled contract to receive one header every time that this function is called and save this header in the internal storage of the bridge. 
This also included a modification in the actual method to receive headers in the same precompiled contract. These methods receive an array of headers to be stored. The modification for this method implies not allow to be called for external users after this RSKIP. 

## Motivation

Bridge precompiled contract includes a function to can receive an array of heads to be stored. This function could be exposed to attacks. To avoid this, some new limits are included. 
Powpeg nodes need to send a group of headers anyway. 
Since one of these new limits is to receive only one header per time, and these limits are needed only for external users, a new method with the new limits is created and the original one was modified to be used only by powpeg nodes. 

## Specification

To implement this change we suggest making the following modifications:

New function:
```
 reveive_header(bytes header) returns (int256 returnValue)
```

Modification in the actual receive_headers function:
After this RSKIP the function needs to return to private. 


### reveive_header limits

1) The size of the parameter received is verified. These limits also exist in receive_headers function. 
2) Header is not previously saved in storage.
3) Time between calls to receive_headers is too short. The time specified between calls to this function is got using the constants of each network. The constant used to compare is named minSecondsBetweenCallsToReceiveHeader.
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

### reveive_headers changes

After the activation of this RSKIP, the function can be called only by powpeg node. This modification is needed in the function that calculates if the method is public or not. This function returns true after RSKIP124, and must return false after this RSKIP.


### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).