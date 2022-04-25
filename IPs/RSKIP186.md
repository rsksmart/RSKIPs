---
rskip: 186
title: Active Federation creation block height registration
description: 
status: Draft
purpose: Sec
author: JD (@josedahlquist)
layer: Core
complexity: 1
created: 2020-11-19
---
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

Additionally, once a Federation migration process finishes, the retiring Federation is removed. This RSKIP proposes to keep the last retiring Federation address in storage, to be used in case there are outstanding funds to migrate after the retiring age.

## Motivation

When a user gets the Federation address they usually interact with a node/service and have to trust this information. This exposes a risk where a man-in-the-middle kind of attack could lead users to send their funds to an invalid address thus losing them.

## Specification

### Storage changes

The Bridge will store three new values:
- activeFederationCreationBlockHeight. (long)
  - Defaults to the existing Federation activation height 
- nextFederationCreationBlockHeight. (long)
  - Defaults to null.
- lastRetiredFederationAddress. (btcAddress)
  - Defaults to null.

### Commit Federation process changes

When a new Federation is committed and the `commit_federation` event is triggered, the Bridge will store the block height in `nextFederationCreationBlockHeight`.

When the Federation is committed, after setting the retiring Federation as the previous one, Bridge will set `lastRetiredFederationAddress` using the retiring Federation as its value.

### updateCollections method changes

The updateCollections method is responsible for the supporting operations of the Bridge, as such, this method now will have to check if a new Federation became Active.

When a new Federation activates, the Bridge will replace `activeFederationCreationBlockHeight` with the value in `nextFederationCreationBlockHeight` and clear the value of `nextFederationCreationBlockHeight`.

### registerBtcTransaction method changes

During the peg-in process, the Bridge evaluates which kind of BTC transaction it is requested to register. To detect if it's a migration, it checks if the sender is the retiring Federation, and if the recipient is the active Federation. Add to this check that the sender could be the retiring Federation, or the last Retired Federation.

### getActiveFederationCreationBlockHeight method

```
getActiveFederationCreationBlockHeight() returns uint256
```

This method will return the Active Federation block height. To do this, it should:
- check if there is a value for `nextFederationCreationBlockHeight`.
  - If so, it should verify if the current block is higher than or equals to `nextFederationCreationBlockHeight` + `federationActivationAge` (Mainnet: 18500, Testnet: 60, Regtest: 10).
    - If so, it should return `nextFederationCreationBlockHeight`.
- Otherwise, it should return the value stored in `activeFederationCreationBlockHeight`.

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
