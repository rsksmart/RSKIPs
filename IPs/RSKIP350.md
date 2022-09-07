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

`epochLength` is defined as 120 blocks. 

These new methods are created for the bridge contract:

* `getOpenPegoutEpoch() returns (uint32)`
* `getLastPegoutRequestId() returns (uint256)`
* `getPendingPegoutState(bytes32 requestId) returns (bytes)`
* `acceptPendingPegouts(bytes32 requestId,uint32 maxEntries)`
* `rejectPendingPegouts(bytes32 requestId,uint32 maxEntries)`

All methods can be called by EOAs or by the `CALL` opcodes.

Two storage variables are created:

* `lastRequestId`
* `openEpoch`

Both are missing (zero) at the begining.
The methods `getOpenPegoutEpoch()` and `getLastPegoutRequestId()` return the values contained in the respective storage fields (`openEpoch` and `lastRequestId`).

## Changes to releaseBtc

First, when the method `releaseBtc()` is called, the internal method `epochEndCheck` is called.
Second, if `openEpoch` is lower or equal to the block height divisible by `epochLength`, then its value is replaced by the new epoch value.

Afterwards, the peg-out request is stored temporarily in a new map named `pendingPegoutRequests`. The map is kept active for `epochLength`.

Each peg out request is identified its by Keccak(`rskTxHash`,`lastRequestId`) where `lastRequestId` represents the id of the last request. It will be zero (32 zero bytes) if it is the first request in the map. The field `rskTxHash` is the transaction where this peg-out originated.

The cost of `releaseBtc()` is increased to 50000, to account for up to 7 rejections. On success, the `releaseBtc()` will emit an event `pendingPegout(requestId,rskTxHash)`.

The `lastRequestId` field is updated with the `requestId` of the request inserted in the map.

Each entry in the map consist of:

* The peg-out request data (`senderAddress`, `amount`, `rskTxHash`)
* `voted`: The list of pegnatories that have already acceptd or rejected this peg-out request, as a bit-vector.
* `acceptanceCount`: the current number of acceptances
* `rejectionCount`: the current number of rejections
* `previous`: the `requestId` of the previous peg-out request in the linked-list, or zero if none.
* `requestEpoch`: the epoch number when this peg-out request was added to the map for the first time.

This forms a single-linked list of pending requests.

Note that the entry does not stores the Bitcoin destination address, but the RSK sender address, to enable reimbursements.
 
Initially, the fields `acceptanceCount`, `rejectionCount` and `voted` are set to zero. The field `previous` is set to the previous `requestId`. The field `requestEpoch` is filled with the current epoch number (block heigh divided by `epochLength`).

To help us define the external methods, we first define 3 internal methods `accept(requestId)` and `reject(requestId)`, `cancel(requestId)`.


## accept
When the `requestId` exists in the map, and the `msg.sender` is a valid pegnatory and the pegnatory index is not set in `voted`, and the reject or accept thresholds have not been reached:

1. The number of acceptances for this peg-out request (`acceptanceCount`) is incremented by one
2. The pegnatory index is set in its `voted` bit-vector.

## reject
When the `requestId` exists in the map, and the `msg.sender` is a valid pegnatory and the pegnatory index is not set in `voted`, and the reject or accept thresholds have not been reached:

1. The number of rejections for this peg-out (`rejectionCount`) request is incremented by one
2. The pegnatory index is set in its `voted` bit-vector.


## cancel
If the peg-out `requestId` request exists:

* the storace cells used are zeroed so they are removed from storage 
* an event `pegOutCancelled(requestId, rskTxHash)` is emitted 
* the amount locked in the peg-out request removed is returned to the `senderAddress`, using direct account balance modification. 

## epochEndCheck

If the current block number divided by `epochLength` is lower or equal to `openEpoch`, then this method returns immediately.
Otherwise, all elements in the map are scanned using the linked list starting from `lastRequestId`.  If for a peg-out request the signatories threshold of acceptance is reached, the request is moved to the peg-out queue, and an event `pegOutAccepted(requestId, rskTxHash)` is emitted. When moving to the request to the release request queue, the Bitcoin destination address should be computed from the stored `senderAddress`.

All requests that have not reached the peg-out threshold, but their `requestEpoch` curresponds to the previous epoch are moved to a new linked list that will replace the current one (this is done by updating the `previous` fields). Entries are moved presserving their order. The remaining entries are cancelled with `cancel`.
The `lastRequestId` field is set to 32 zero bytes if no request was moved from the previous epoch, or it will contain the `requestId` of the last moved entry.
This method is called by `updateCollections`, `releaseBtc`, `acceptPendingPegouts` and `rejectPendingPegouts`.

## acceptPendingPegouts

First, the `epochEndCheck` method is called.

This method enables the acceptance of all pending peg-out requests up to a certain pending request, and upto a maximum number of pending requests scanned given as argument. The internal `accept` method is called for each of the scanned entries.
The method does not return any value.
The gas cost of the `acceptPendingPegouts` method is set to 5000*maxEntries.

## rejectPendingPegouts

First, the `epochEndCheck` method is called.

This method enables the rejection of all pending peg-out requests up to a certain pending request, and upto a maximum number of pending requests scanned given as argument. The internal `reject` method is called for each of the scanned entries.

The method does not return any value.
The gas cost of the `rejectPendingPegouts` method is set to 5000*maxEntries.

## getPendingPegoutState

This method returns the peg-out request entry serialized. If the `requestId` does not exists, the returned data is empty.

## Internal Storage

Entries in the `pendingPegoutRequests` are stored as independent cells in the Bridge contract storage. To derive the storage address, the message `("pendingAcceptance-" + Hex(requestId))` is hashed with Keccak256. The function Hex() returns the 64-byte lowercase hexadecimal representation of the hash without the "0x" prefix.

The peg-out request payload consist of:

* `senderAddress`
* `amount`
* `rskTxHash`
* `acceptanceCount`
* `rejectionCount`
* `voted`
* `previous`
* `requestEpoch`

The `acceptanceCount` and `rejectionCount` fields are stored as an `int16`.
The `voted ` field is a bit-vector of fixed size 16 bytes (supports a maximum of 128 pegnatories). The most significant bit of the first byte corresponds to the first pegnatory.
The `previous` is an `uint256`. All fields are fixed-length.

## Migration
During powpeg migration, when the new federation is activated, if there are peg-out request outstanding in the map then:

* All entries have their fields `acceptanceCount`, `rejectionCount` and `voted` zeroed. The fields are zeroed even if they had already reached their acceptance or rejection thresholds.
* The event `pendingPegouts(lastRequestId)` is emmitted.

The new powpeg must approve or reject the prexisting set of peg-out requests. 

## Peg-out Batching

The batch creation time is redefined in terms of epochs, specifically 3 epochs. Every 3 epochs the bridge will attempt to create a new batch. 

## Peg-out Fee Estimation

Peg-out fees will be estimated according to the number of transactions queued in the batch, as it is today, and not conting the transactions that are still in the `pendingPegoutRequests` map.

# Rationale

This proposal is deisgned to enable batch accepts and rejections, because forcing individual votes can be very costly to pegnatories in terms of gas spent. To enalbe batch votes, it is important that the batch voted cannot change between the time the pegnatory is notified of the batch and the time the vote transaction is included in the blockchain. That's why votes apply to all peg-out requests until a specified point in the batch. An alternative would be to divide the time in two phases, one that builds a batch, and another one for voting. I nthe voting phase, new peg-outs are queued for the following batch. However, this design still has the problem that blockchain reorganizations can change the content of batches, and pegnatories votes should never mutate with block reorganizations. Therefore the votes must include either the id of a chain of peg-outs, or a block hash (which indirectly also fixes all previous peg-out requests). 
 
This proposal ensures that pegnatories will always have at least one full epoch to accept ore reject a certain peg-out request. Peg-out requests issued close to the deadline of the epoch will be moved to the next epoch. The maximum time a peg-out request can be outstanding is two epochs, minus one block. We set `epochLength` to 120, which corresponds approximately to one hour. This gives enough time for pegnatories to perform any automated check, yet it doesn't block user funds for too long. 
 
Redefiniting the batching time in term of epochs allow to synchronize both and that the batch waiting time is not extended by one epoch. However, if a peg-out request is issued close to the end of the epoch prior a batch creation, the peg.out may not receive enough votes and it will be delayed until the net batch creation event.

This proposals has theflexibility to accept/reject all or specific peg-outs to reduce the number of Bridge calls required and hashes transferred. 

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 


# Test Cases

TBD

## Security Considerations

No security risks have been identified related to this change.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
