---
rskip: 350
title: Peg-out Acceptance
description: 
status: Draft
purpose: Usa
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2022-09-06
---
# Peg-out Acceptance

|RSKIP          | 350          |
| :------------ |:-------------|
|**Title**      |Peg-out Acceptance|
|**Created**    |06-SEP-2022 |
|**Author**     |SDL |
|**Purpose**    |Usa |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     ||

# **Abstract**

This RSKIP proposes enables pegnatories to accept specific peg-outs from a peg-out batch, and reject specific peg-outs, or accept/reject all outstanding peg-outs.

# **Motivation**

The bridge contract is not designed to allow the blocking of peg-outs. Pegnatories may be requested to do it. To comply, pegnatories may need to turn off the PowHSM devices. However, the peg-outs are executed as soon as the devices are turned on and connected to the network again. The bridge assumes peg-out transactions are always executed and it cannot cancel a peg-out already commanded. Peg-outs are grouped in batches so that a Bitcoin transaction can contain several peg-outs and therefore selective blocking cannot be performed.

In this proposal, we give pegnatories the capability to accept peg-outs before they are included in the batch and signed.

# **Specification**

When the method `releaseBtc()` is called, the peg-out request is stored temporarily in a new map named `pendingPegoutRequests`. The requests are identified by Keccak(`rskTxHash`,`parentBlockId`) where `parentBlockId ` represent the id of the parent block.

key in the map is the hash of RSK transaction hash where the peg-out request originated. The data in the map consist of:

* The peg-out request data (`senderAddress`, `amount`, `rskTxHash`)
* `voted`: The list of pegnatories that have already acceptd or rejected this peg-out request, as a bit-vector.
* `acceptanceCount`: the current number of acceptances
* `rejectionCount`: the current number of rejections
* `blockNumber`: the block number where this peg-out request was issued.
* `previous`: the id of the previous peg-out request, or zero if none.
* `next`: the id of the next peg-out request, or zero if none.

This forms a linked list of pending requests.

Note that the entry does not stores the Bitcoin destination address, but the RSK sender address, to enable reimbursements.
 
Initially, the fields `acceptanceCount`, `rejectionCount` and `voted` are set to zero and the block height is stored in `blockNumber`.

Five new methods are created for the bridge contract:

* `accept(bytes32 requestId) returns (uint32)`
* `reject(bytes32 requestId) returns (uint32)`
* `cancel(bytes32 requestId) returns (uint32)`
* `acceptUpto(bytes32 requestId,uint32 maxEntries)`
* `rejectUpto(bytes32 requestId,uint32 maxEntries)`

All methods can be called by EOAs or by the `CALL` opcodes.
The cost of `releaseBtc()` is increased to 50000, to account for up to 7 rejections. 


## accept
While the peg-out request is in the `pendingPegoutRequests` map, pegnatories can call the method `accept(requestId)`. When the requestId exists in the map, and the msg.sender is a valid pegnatory and it is the first time the pegnatory accepts or rejects the peg-out request, and the reject or accept thresholds have not been reached:

1. The number of acceptances for this peg-out request (`acceptanceCount`) is incremented by one
2. The pegnatory index is added to its `voted` bit-vector.

In this case the `accept` method then returns 1.
If the msg.sender is not a valid pegnatory, then the method does nothing and returns 2.
If a valid pegnatory calls this method more than once, the following calls are ignored, and the method returns 3.
When the signatories threshold of acceptances is reached, the request is moved to the peg-out queue, un-linked from the embedded link-list, and the method returns 4. When moving to the request to the release request queue, the Bitcoin destination address should be computed from the stored `senderAddress`.
The gas cost of the `accept` method is set to 5000.

## reject
While the peg-out request is in the `pendingPegoutRequests` map, pegnatories can call the method `reject(requestId)`. When the requestId exists in the map, and the msg.sender is a valid pegnatory and it is the first time the pegnatory accepts or rejects the peg-out request, and the reject or accept thresholds have not been reached:

1. The number of rejections for this peg-out (`rejectionCount`) request is incremented by one
2. The pegnatory index is added to its acceptances bit-vector.


In this case the `reject` method then returns 1.
If the msg.sender is not a valid pegnatory, then the method does nothing and returns 2.
If a valid pegnatory calls this method more than once, the following calls are ignored, and the method returns 3.
When the signatories threshold of rejections is reached, the request is cancelled and the method returns 4. The rejection threshold is the number of pegnatories minus the acceptance threshold, plus one.

The gas cost of the `reject` method is set to 5000.

## cancel
If a user calls `cancel(requestId)` and the peg-out requestId request exists, and the current block height is higher than the block number stored in the map plus 3000, the peg-out request is removed from the `pendingPegoutRequests` map, and it is unlinked from the linked-lisst. This involves eventually modifying the previous and next request records of point at each other. The amount locked in the peg-out request removed is returned to the `senderAddress`, using direct account balance modification. In this case, the returned value is 1. If the peg-out request does not exist, the returned value is 2. If the block heght deadline has not been reached, the returned value is 3.

The gas cost of `cancel` is set to 15000. 
## acceptUpto

This method enables the acceptance of all pending peg-out requests up to a certain pending request, and upto a maximum number of pending requests scanned given as argument.
The method does not return any value.
The gas cost of the `acceptUpto` method is set to 5000*maxEntries.

## rejectUpto

This method enables the rejection of all pending peg-out requests up to a certain pending request, and upto a maximum number of pending requests scanned given as argument.
The method does not return any value.
The gas cost of the `rejectUpto` method is set to 5000*maxEntries.

## Internal Storage

Entries in the `pendingPegoutRequests` are stored as independent cells in the Bridge contract storage. To derive the storage address, the message `("pendingacceptance-" + Hex(requestId))` is hashed with Keccak256. The function Hex() returns the 64-byte lowercase hexadecimal representation of the hash without the "0x" prefix.

The peg-out request payload consist of:

* `senderAddress`
* `amount`
* `rskTxHash`
* `blockNumber`
* `acceptanceCount`
* `rejectionCount`
* `voted`
* `previous`
* `next`

The `acceptanceCount` and `rejectionCount` fields are stored as an `int16`.
The `voted ` field is a bit-vector of fixed size 16 bytes (supports a maximum of 128 pegnatories).
The `blockNumber` is stored as an `int32`.
The `previous` and `next` fields are stored as `uint256`. 


# Rationale

The flexibility to accept/reject all or specific peg-outs reduces the number of Bridge calls required and hashes transferred.

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 


# Test Cases

TBD

## Security Considerations

No security risks have been identified related to this change.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
