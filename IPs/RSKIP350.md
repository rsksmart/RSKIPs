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

These new methods are created for the bridge contract:

* `getPendingPegoutState(bytes32 requestId) returns (bytes)`
* `cancelPendingPegout(bytes32 requestId) returns (uint32)`
* `acceptPendingPegouts(bytes32 requestId,uint32 maxEntries)`
* `rejectPendingPegouts(bytes32 requestId,uint32 maxEntries)`

All methods can be called by EOAs or by the `CALL` opcodes.
The cost of `releaseBtc()` is increased to 50000, to account for up to 7 rejections. 

To help us define these external methods, we first define 2 internal methods `accept(requestId)` and `reject(requestId)`.

## accept
When the `requestId` exists in the map, and the msg.sender is a valid pegnatory and the pegnatory index is not set in `voted`, and the reject or accept thresholds have not been reached:

1. The number of acceptances for this peg-out request (`acceptanceCount`) is incremented by one
2. The pegnatory index is set in its `voted` bit-vector.

When the signatories threshold of acceptances is reached, the request is moved to the peg-out queue, un-linked from the embedded link-list, and an event `pegOutAccepted(requestId, rskTxHash)` is emitted. When moving to the request to the release request queue, the Bitcoin destination address should be computed from the stored `senderAddress`.

## reject
When the `requestId` exists in the map, and the msg.sender is a valid pegnatory and the pegnatory index is not set in `voted`, and the reject or accept thresholds have not been reached:

1. The number of rejections for this peg-out (`rejectionCount`) request is incremented by one
2. The pegnatory index is set in its `voted` bit-vector.

When the signatories threshold of rejections is reached, the request is cancelled (in a similar way to the cancel method). The rejection threshold is the number of pegnatories minus the acceptance threshold, plus one.


## cancel
If a user calls `cancel(requestId)` and the peg-out requestId request exists, and the current block height is higher than the block number stored in the map plus 3000, the peg-out request is removed from the `pendingPegoutRequests` map, and it is unlinked from the linked-list. This involves eventually modifying the previous and next request records of point at each other. 
Also, an event `pegOutCancelled(requestId, rskTxHash)` is emitted.
The amount locked in the peg-out request removed is returned to the `senderAddress`, using direct account balance modification. In this case, the returned value is 1. If the peg-out request does not exist, the returned value is 2. If the block heght deadline has not been reached, the returned value is 3.

The gas cost of `cancel` is set to 15000. 

## acceptPendingPegouts

This method enables the acceptance of all pending peg-out requests up to a certain pending request, and upto a maximum number of pending requests scanned given as argument. The internal `accept` method is called for each of the scanned entries.
The method does not return any value.
The gas cost of the `acceptPendingPegouts` method is set to 5000*maxEntries.

## rejectPendingPegouts

This method enables the rejection of all pending peg-out requests up to a certain pending request, and upto a maximum number of pending requests scanned given as argument. The internal `reject` method is called for each of the scanned entries.

The method does not return any value.
The gas cost of the `rejectPendingPegouts` method is set to 5000*maxEntries.

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
