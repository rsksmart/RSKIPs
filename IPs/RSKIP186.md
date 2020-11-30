# Active Federation creation block height registration

|RSKIP          |186           |
| :------------ |:-------------|
|**Title**      |Active Federation creation block height registration |
|**Created**    |19-NOV-20 |
|**Author**     |JD |
|**Purpose**    |Sec |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft |

## Abstract

When the Federation change voting process finishes, the Bridge emmits a `commit_federation` event.
This event can be used to provide a blockchain proof of the Federation address when requested. Nowadays doing this would imply navigating the blockchain to determine when it was emmited, or constantly monitoring said events.

This RSKIP proposes adding on-chain information to easily fetch the block where the Federation was changed for the last time. With the block, a system could provide proof that it belongs to the mainchain.

## Motivation

When a user gets the Federation address they usually interact with a node/service and have to trust this information. This exposes a risk where a man-in-the-middle kind of attack could lead users to send their funds to an invalid address thus losing them.

## Specification

### Storage changes

The Bridge will store two new values:
- activeFederationCreationBlockHeight. (long)
  - Defaults to the existing Federation activation height 
- nextFederationCreationBlockHeight. (long)
  - Defaults to the existing Federation activation height

### Commit Federation process changes

When a new Federation is committed and the `commit_federation` event is triggered, the Bridge will store the block height in `nextFederationCreationBlockHeight`.

When a new Federation activates, the Bridge will replace `activeFederationCreationBlockHeight` with the value in `nextFederationCreationBlockHeight`.


This means that when the `commit_federation` event is emmited, the Bridge should store an additional value (e.g. `nextFederationCreationBlockHeight`) and when the new Federation becomes the active one, then it will update the definitive value. It is important not to replace the active value before the Federation actually changes (in Mainnet this happens after 18500 blocks).

When this RSKIP gets implemented it should include an initial value with the last Federation change in each network.

### getActiveFederationCreationBlockHeight method

```
getActiveFederationCreationBlockHeight() returns uint256
```

This method will return the value stored in `activeFederationCreationBlockHeight`.

## Rationale

### Scope

This RSKIP only contemplates the consensus changes, this means that the Bridge will expose a method to get the block, but not a way to generate the merkle proof of said block's inclusion in the blockchain. Once the consensus changes are in place anyone (e.g. the rskj node) could provide a method to fetch the Federation address with merkleproof.

### Federation activation

When a new Federation is committed, this doesn't mean the Federation changes immediately, for this to happen a number of blocks has to be mined (18500 in Mainnet), this is the reason why the RSKIP proposes to store two values. It's important that the block height returned corresponds to the active Federation (return with the Bridge method `getFederationAddress()`).

### Default Federation activation block value

There is a risk that in between the node deploy and the consensus activation an unexpected Federation change occurs. If this would happen, the source code could point to an older active Federation creation block.
The only way to solve this would be to ensure that the Federation is not changed once the code is deployed and before the network upgrade is active.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
