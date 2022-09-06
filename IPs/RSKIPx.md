---
rskip: 
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

|RSKIP          |           |
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

This RSKIP proposes enables pegnatories to accept specific peg-outs from a peg-out batch, and reject the rest.

# **Motivation**

The bridge contract is not designed to allow the blocking of peg-outs. Pegnatories may be requested to do it. To comply, pegnatories may need to turn off the PowHSM devices. However, the peg-outs are  executed as soon as the devices are turned on and connected to the network again. The bridge assumes peg-out transactions are aways executed and it cannot cancel a peg-out already commanded. Peg-outs are grouped in batches so that a Bitcoin transaction can contain several peg-outs and therefore selective blocking cannot be performed.

In this proposal, we give pegnatories the capability to accept peg-outs before they are included in the batch and signed.

# **Specification**

When the method `releaseBtc()` is called, the peg-out request is stored temporarily in a new map named `pendingApprovals`. The key in the map is the RSK transaction hash where the peg-out request originated. The data in the map consist of:

* The peg-out request data
* The list of pegnatories that have already approved this peg-out request, as a bit-vector.
* The current number of approvals
* The block number where this peg-out request was issued.
 
Two new methods are created for the bridge contract:

* `uint32 approve(bytes32 RSKtxHash)`
* `uint32 cancel(bytes32 RSKtxHash)`

While the peg-out request is in the `pendingApprovals` map, pegnatories can call the method `approve(RSKtxHash)`. When the RSKtxHash exists in the map, and the pegnatory signature is valid and it is the first time the pegnatory approves the peg-out request, and the threshold has not been reached:

1. The number of approvals for this peg-out request is incremented by one
2. The pegnatory index is added to its approvals bit-vector.
3. The block height is stored.

In this case the `approve` method then returns 1.
If the pegnatory signature is invalid (not a member of the powpeg), then the method does nothing and returns 2.
If a valid pegnatory calls this method more than once, the following calls are ignored, and the method returns 3.
When the signatories threshold of approvals is reached, the request is moved to the peg-out queue, and the method returns 4.

If a user calls `cancel(RSKtxHash)` and the peg-out RSKtxHash request exists, and the current block height is higher than the block number stored in the map plus 3000, the peg-out request is removed from the `pendingApprovals` map, and the amount locked for peg-out is returned to the same EOA it came from, using direct account balance modification. In this case, the returned value is 1. If the peg-out request does not exist, the returned value is 2. If the block heght deadline has not been reached, the returned value is 3.

The Gas cost of each method call is 23000, irrespective of the return value. 

## Internal Storage

Entries in the `pendingApprovals` are stored as independent cells in the Bridge contract storage. To derive the storage address, the message `("pendingApproval-" + Hex(RSKtxHash))` is hashed with Keccak256. The function Hex() returns the 64-byte lowercase hexadecimal representation of the hash without the "0x" prefix.

The peg-out request payload consist of:

* `destination`
* `amount`
* `rskTxHash`
* `blockNumber`
* `approvalCount`
* `approvals`

The `approvalCount` is stored as an `int16`.
The `approvals` field is a bit-vector of fixed size 32 bytes (supports a maximum of 256 pegnatories).
The `blockNumber` is stored as an `int32`.


# Rationale

TBD.

# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. 


# Test Cases

TBD

## Security Considerations

No security risks have been identified related to this change.


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
