---
rskip: 135
title: Managing BridgeMaster Federation Members
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2016-11-25
---
# Managing BridgeMaster Federation Members

|RSKIP          |135           |
| :------------ |:-------------|
|**Title**      |Managing BridgeMaster Federation Members|
|**Created**    |25-NOV-2016 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

This document describes the procedure to add or remove bridge federators, and the procedure which federators can use to change their private keys. This document does not address the "extended" federation, which provides other services. 

# Discussion

As changing the federation is an expensive process, the system must allow to apply several changes to a federation.

The design is based on the following ideas:

* The resign process is unilateral. Nobody else needs to approve that a federator resigns.

* The security critical operations are so simple that a simple confirmation message can be displayed in a hardware wallet display. For example: to add a federator, the display can show the message "Are you sure to add the federator X with public key hash ZZZ". Then the manager of the federation can check that the displayed hash corresponds to a hash displayed on the (maybe insecure) PC screen, together with the (maybe insecure) smart-phone. 

* Also the hardware token could itself to check the validity of public keys by accessing a PKI infrastructure.

# **Specification**

BridgeMaster Federators are the federators that are involved in the multi-signature of the Bitcoin bridge. From now on, "federators" is limited to this kind of federators members.

One of the drawbacks of current design is that the bridge contract stores the public keys of the federators. In the future, and via hard-forks, the federators may control other bridge systems.

Therefore is good practice to re-factor the BridgeMaster into at least three contracts. 

* BridgeFedFront

* BridgeFed (one or more active)

* BridgeMaster

All services provided by these contracts must respect the ABI interface.

The following diagram shows how different contracts interact.

## All signatures must sign also the federation id to prevent replay attacks. For example, to add a federator, the message that it signed is: < NewFedID | ADD | PubKey | name >

## BridgeFedFront

The BridgeFedFront contract will provide the following services:

**1. getCurrentFederation() → bridgeFed :BridgeFed**

Get the federation that is active and recommended now to send funds to.

**2. getActiveFederations() → array of BridgeFed**

Returns the list of federations that can still receive funds, but funds may be moving to the current fed.

**3. getNewFederation() → bridgeFed :BridgeFed**

The first time called by a member, it creates a new Fed clone of the current fed into a new BridgeFed contract. This is a candidate federation. The second time it’s called by a member it returns the latest newly created Fed that is still inactive. Once the new fed is activated, any call to getNewFederation() will result in the creation of yet another fed.

## BridgeFed

Each federator is represented by an id. 

**1 .getMemberCount() → uint count **

Returns the number of active federators in this federation.

**2. getMemberPubKeys() → array of public keys**

Returns an array of all the public keys of this federation.

**3. getMemberPubKey(uint id) → PublicKey**

Returns a public key of a specific member

**4. getMemberName(uint id) → String**

Returns the member name by id.

**5. getMemberId(PublicKey key) → uid**

Return the public key of a single member of the federation, by mid.

**6. getPrevFederation() → BridgeFed**

Returns the contract that controls the federation that gave birth to this federation.

**7. voteAddMember(PublicKey newPubKey, String memberName, Signature NewPubKey, Signature prevFedMemberSig)**

A previous federation member can vote to add a new member to this federation. Internally, for each (newPubKey,memberName**)** indicated, a vote is recorded in the associated poll. Every time a new member votes, the poll vote count is increased. If a poll vote count surpass the voteThreshold (50% of voters), then member is added to the FedBridge (getMemberCount() and getMemberPubKeys() will inform of the new member). There can be two active polls with the same newPubKey but different memberName fields. This is to prevent the first voter to set the memberName to his choice. So it’s acceptable that there are two active polls. However, only ONE of them can be added, the other represents a mistake or an attack, and there can’t be two federators with the same public key. There can be two members with the same memberName but different public keys. This is acceptable.

**8. voteRemoveMember(PublicKey aPubKey,Signature prevFedMemberSig)**

A previous federation member can vote to remove a new member of this federation. Internally, for each newPubKey indicated, a remove vote is recorded in the associated poll. Every time a new member votes, the poll vote count is increased. If a poll vote count surpass the voteThreshold (50% of voters), then the member is removed from the FedBridge (getMemberCount() and getMemberPubKeys() will inform of the new member). It must be noted that a member can be added and then removed, or vice-versa. Even multiple times. If the member changes his public key using changeMemberPubKey() 

**9. voteInvalidate(Signature prevFedMemberSig)**

This method can be used to invalidate a federation before it goes into effect. To invalidate a new federation, the majority of the previous federation members must vote using this method. 

**10. voteAccept(Signature newFedMemberSig)**

The new federation will be automatically accepted N hours after the last time a new federation member was added or removed. However, the new federation members can choose to accelerate the acceptance process by voting to accept this federation. Every time voteAccept, is called, the majority threshold is checked (against the current set of members). 

**11. changeMemberPubKey(PublicKey newPubKey, Signature NewPubKey, Signature oldKeySig) → boolean success**

This method allows a member to change its own public key. This change only occurs when the federation is activated, and only if the federation has not been removed. This call does not require voting. The new public key enters into action when this federation is in effect. All votes are still cast using the old key. The new public key cannot be duplicated.

**12. resign(Signature sig) → boolean success**

This allows a member to resign to the federation. All but the last member can resign. If the last member tries to resign, the function returns false. The resign method almost purely informative. It does not affect how the new federation votes are cast. If a resign call is received in a parent federation after a new child federation is created, it does not affect the child vote.

**13.  getBlockCountToActivation() → uint**

This function returns the number of blocks that must pass until this federation is automatically activated. When a federation has already been activated, it returns 0.

When a federation is activated, it will notify the BridgeFedFront contract. The BridgeFedFront contract will notify the BridgeMaster Contract.The BridgeMaster contract will start the process of migrating funds, and when the process is over it will call the method disposeFed(f) on the federation that has no more access to funds, nor it is allowed to receive more funds.

 

**14. GetFedAddress() → String**

This method is only available when this federation is activated. Returns the Bitcoin address (format to be determined) of this federation

**15. SetFedAddress(addr String)**

This method can be called only by the BridgeMaster contract to set the Bitcoin address for this federation, after the federation is activated. Once the value is set, it cannot be changed anymore.

## BridgeMaster

The BridgeMaster will hold a pointer to the BridgeFedFront contract.

The BridgeMaster will keep active any federation that still has funds associated, but will try to move any UTXO to the latest active federation. Once a new active federation appears, the BridgeMaster will accept payments to old federations for a period of M hours (configurable). Afterwards, any received bitcoins will not be accepted and those amounts will be locked forever. Once an old federation has no more UTXOs associated and the M hours period has elapsed, it will be removed from BridgeFedFront and the BridgeMaster will dispose all information related to it.

**Moving the Funds to the new Federation**

Funds are moved immediately after the new federation is activated.

In the future there will be two types of federation transitions. Emergency transitions, where all funds are moved at once, and normal transitions, where for a lapse funds are moved only when generating new change addresses and after the lapse is over the remaining funds are moved all at once.

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).